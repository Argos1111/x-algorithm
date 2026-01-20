# X（Twitter）レコメンデーションアルゴリズム 詳細分析

## 概要

このアルゴリズムは、**2段階のパイプライン**で構成されています：
1. **候補取得（Retrieval）**: Thunder（ネットワーク内）とPhoenix（ネットワーク外）から候補を収集
2. **ランキング（Ranking）**: Grokトランスフォーマーで19種類のエンゲージメント確率を予測し、重み付けスコアを計算

---

## スコアリングの仕組み

### 1. PhoenixScorer（MLモデルによる予測）

`home-mixer/scorers/phoenix_scorer.rs:129-151` で定義された19種類のアクション確率を予測：

**ポジティブなアクション（スコア増加）:**

| アクション | 説明 |
|------------|------|
| `favorite_score` | いいねする確率 |
| `reply_score` | リプライする確率 |
| `retweet_score` | リツイートする確率 |
| `quote_score` | 引用ツイートする確率 |
| `share_score` | シェアする確率 |
| `share_via_dm_score` | DMでシェアする確率 |
| `share_via_copy_link_score` | リンクコピーでシェアする確率 |
| `click_score` | ツイートをクリックする確率 |
| `profile_click_score` | プロフィールをクリックする確率 |
| `photo_expand_score` | 画像を拡大する確率 |
| `vqv_score` | 動画を品質視聴する確率 |
| `dwell_score` | 一定時間滞在する確率 |
| `dwell_time` | 予測滞在時間（連続値） |
| `quoted_click_score` | 引用元をクリックする確率 |
| `follow_author_score` | 著者をフォローする確率 |

**ネガティブなアクション（スコア減少）:**

| アクション | 説明 |
|------------|------|
| `not_interested_score` | 「興味なし」を選択する確率 |
| `block_author_score` | 著者をブロックする確率 |
| `mute_author_score` | 著者をミュートする確率 |
| `report_score` | ツイートを報告する確率 |

### 2. WeightedScorer（重み付け合計）

`home-mixer/scorers/weighted_scorer.rs:44-70` で実装：

```rust
combined_score =
    FAVORITE_WEIGHT × P(favorite) +
    REPLY_WEIGHT × P(reply) +
    RETWEET_WEIGHT × P(retweet) +
    QUOTE_WEIGHT × P(quote) +
    SHARE_WEIGHT × P(share) +
    // ... その他のポジティブアクション ...
    NOT_INTERESTED_WEIGHT × P(not_interested) +  // 負の重み
    BLOCK_AUTHOR_WEIGHT × P(block_author) +      // 負の重み
    MUTE_AUTHOR_WEIGHT × P(mute_author) +        // 負の重み
    REPORT_WEIGHT × P(report)                     // 負の重み
```

**重要**: 実際の重み値は `params` モジュールで定義されていますが、「セキュリティ上の理由」でオープンソースには含まれていません。

### 3. AuthorDiversityScorer（著者多様性）

`home-mixer/scorers/author_diversity_scorer.rs:29-31`:

```rust
multiplier(position) = (1 - floor) × decay_factor^position + floor
```

同じ著者の投稿が連続して表示されないよう、**指数関数的減衰**を適用します。2番目以降の同一著者の投稿はスコアが下がります。

### 4. OONScorer（ネットワーク外ペナルティ）

`home-mixer/scorers/oon_scorer.rs:20-22`:

```rust
match c.in_network {
    Some(false) => base_score * OON_WEIGHT_FACTOR,  // 0 < factor < 1
    _ => base_score,  // フォロー中は減点なし
}
```

フォローしていないアカウント（out-of-network）の投稿は、`OON_WEIGHT_FACTOR` で減点されます。

---

## 優遇される投稿の傾向

### 強く優遇される投稿

1. **高いエンゲージメントが予測される投稿**
   - いいね、リツイート、リプライ、引用ツイートされやすい投稿
   - シェア（DM、リンクコピー含む）されやすい投稿

2. **フォロー中のアカウントの投稿（in-network）**
   - ネットワーク外の投稿より優先される

3. **ユーザーの興味を引く投稿**
   - プロフィールクリックを誘発する投稿
   - 長い滞在時間が予測される投稿
   - フォローにつながりそうな投稿

4. **メディアコンテンツ**
   - 画像を拡大して見たくなる投稿
   - 一定以上の長さの動画（`MIN_VIDEO_DURATION_MS`以上）
     - VQV（Video Quality View）スコアは動画が一定長以上の場合のみ適用 (`weighted_scorer.rs:72-81`)

### MLモデルの特徴（recsys_model.py）

Grokベースのトランスフォーマーモデルは以下を考慮：

1. **ユーザー埋め込み**: ユーザーの特徴をハッシュベースで表現
2. **履歴埋め込み**: 過去のエンゲージメント履歴（投稿、著者、アクション、表示場所）
3. **候補埋め込み**: 候補投稿と著者の特徴

重要な設計（`grok.py:39-71`）:

```python
# 候補間の相互参照を防ぐアテンションマスク
# 各候補は user+history のみを参照し、他の候補は参照しない
# → 独立したスコアリングを保証
```

---

## 不遇される投稿の傾向

### 明示的にペナルティを受ける投稿

1. **ネガティブなアクションが予測される投稿**
   - 「興味なし」を選ばれそうな投稿
   - ブロック・ミュートされそうな投稿
   - 報告されそうな投稿

2. **フィルタで除外される投稿**

   | フィルタ | 除外対象 |
   |----------|----------|
   | `AgeFilter` | `MAX_POST_AGE`秒より古い投稿 |
   | `SelfTweetFilter` | 自分自身の投稿 |
   | `RetweetDeduplicationFilter` | 重複リツイート |
   | `AuthorSocialgraphFilter` | ブロック/ミュートした著者の投稿 |
   | `MutedKeywordFilter` | ミュートしたキーワードを含む投稿 |
   | `PreviouslySeenPostsFilter` | 既に見た投稿 |
   | `VFFilter` | 削除済み/スパム/暴力的コンテンツ |

3. **多様性ペナルティを受ける投稿**
   - 同じ著者の2番目以降の投稿（指数関数的減衰）

4. **ネットワーク外の投稿**
   - フォローしていないアカウントの投稿は `OON_WEIGHT_FACTOR` で減点

---

## 最終スコア計算式

```
final_score = weighted_score
            × author_diversity_multiplier
            × (in_network ? 1.0 : OON_WEIGHT_FACTOR)
```

その後 `TopKScoreSelector` が最終スコアでソートし、上位K件を選択します。

---

## パイプライン実行フロー

```
1. QUERY HYDRATION
   └─ UserActionSeqQueryHydrator: ユーザーエンゲージメント履歴を取得
   └─ UserFeaturesQueryHydrator: フォローリスト、設定を取得

2. CANDIDATE SOURCING (並列)
   └─ PhoenixSource: ML検索（ネットワーク外）
   └─ ThunderSource: フォロー中アカウントの投稿

3. CANDIDATE HYDRATION (並列)
   └─ InNetworkCandidateHydrator
   └─ CoreDataCandidateHydrator
   └─ VideoDurationCandidateHydrator
   └─ SubscriptionHydrator
   └─ GizmoduckCandidateHydrator

4. PRE-SCORING FILTERS (順次)
   └─ DropDuplicatesFilter
   └─ CoreDataHydrationFilter
   └─ AgeFilter (MAX_POST_AGE秒)
   └─ SelfTweetFilter
   └─ RetweetDeduplicationFilter
   └─ IneligibleSubscriptionFilter
   └─ PreviouslySeenPostsFilter
   └─ PreviouslyServedPostsFilter
   └─ MutedKeywordFilter
   └─ AuthorSocialgraphFilter

5. SCORING (順次) ← ランキングはここで行われる
   └─ PhoenixScorer (ML予測)
   └─ WeightedScorer (19スコアを合成)
   └─ AuthorDiversityScorer (重複著者を減衰)
   └─ OONScorer (ネットワーク外を減点)

6. SELECTION
   └─ TopKScoreSelector (スコア順でソート、上位K件を選択)

7. POST-SELECTION HYDRATION
   └─ VFCandidateHydrator (可視性フィルタリング)

8. POST-SELECTION FILTERS
   └─ VFFilter (削除済み/スパム/暴力コンテンツを除外)
   └─ DedupConversationFilter (スレッド重複排除)

9. SIDE EFFECTS
   └─ CacheRequestInfoSideEffect (将来使用のためログ記録)
```

---

## 非公開のパラメータ

`home-mixer/lib.rs:5` によると、以下のモジュールはオープンソースには含まれていません：

- `params` - 全ての重み係数
- `clients` - 外部サービスクライアント
- `util` - スコア正規化など

具体的に非公開の重み：

- `FAVORITE_WEIGHT`, `REPLY_WEIGHT`, `RETWEET_WEIGHT` など全19種類のアクション重み
- `OON_WEIGHT_FACTOR` (ネットワーク外減点率)
- `AUTHOR_DIVERSITY_DECAY`, `AUTHOR_DIVERSITY_FLOOR` (著者多様性パラメータ)
- `MIN_VIDEO_DURATION_MS` (VQV適用の最小動画長)
- `MAX_POST_AGE` (投稿の最大経過時間)

---

## まとめ

### バズりやすい投稿の特徴

- フォローしているアカウントからの投稿
- いいね・リツイート・リプライを誘発しやすいコンテンツ
- シェアされやすいコンテンツ
- 一定以上の長さの動画を含む投稿
- ユーザーが長く滞在する、深く読み込むようなコンテンツ
- フォローにつながるような著者の投稿

### 表示されにくい投稿の特徴

- フォローしていないアカウントの投稿
- ネガティブなリアクション（ブロック、ミュート、報告）を誘発しそうな投稿
- 同じ著者からの連続投稿
- 古い投稿
- 既に見た投稿
