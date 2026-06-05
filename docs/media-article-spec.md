# SalesRadar オウンドメディア（コラム）記事｜CMS スキーマ & ページ仕様

関連ワイヤーフレーム：

- 記事内 監修者コンポーネント：`sr-media-author-wire.html`
- 監修者プロフィールページ：`sr-media-author-profile-wire.html`

データソース：microCMS（ヘッドレスCMS）の `article` コンテンツAPI。

---

## 1. 概要

BtoB営業・マーケティング領域のオウンドメディア（コラム）記事。microCMS で記事・カテゴリ・著者・監修者を管理し、フロントは配信APIから取得して描画（SSG/ISR 推奨）。

- 記事URL：`/column/{slug}/`（本文中の内部リンクが `https://radar.futurewoods.co.jp/column/{slug}` 形式）
- 著者と監修者は**別概念**（§3）：
  - `author`（執筆・発行主体）… 例「SalesRadar編集部」＝**組織（Organization）**
  - `supervisor`（記事の監修者）… 例「猪俣詢」＝**個人（Person）／役職あり**
- E-E-A-T 強化のため、著者・監修者・発行元・パンくず・FAQ を構造化データ（JSON-LD）で出力（§7）。

---

## 2. コンテンツモデル：`article`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `id` | string | （system） | microCMS コンテンツID |
| `createdAt` / `updatedAt` | datetime | （system） | 作成・更新（システム） |
| `publishedAt` | datetime | （system） | 公開日時。記事の「公開日」表示・`datePublished` に使用 |
| `revisedAt` | datetime | （system） | 最終改訂日時。「最終更新日」表示・`dateModified` に使用 |
| `title` | string | ✓ | 記事タイトル。`<h1>`・`<title>`・OGP title |
| `slug` | string | ✓ | URL識別子。`/column/{slug}/` |
| `description` | textArea | ✓ | メタディスクリプション・OGP description・一覧の抜粋。120〜160字目安 |
| `content` | richEditor(HTML) | ✓ | 本文HTML。見出しID・画像・表・内部リンク・FAQを含む（§4） |
| `featuredImage` | image | ✓ | アイキャッチ。**1200×630**（OGP/Twitter Card 兼用）。`url/width/height` |
| `author` | reference → `author` | ✓ | 執筆・発行主体（§3.1） |
| `supervisor` | reference → `supervisor` | – | 記事の監修者（§3.2）。任意 |
| `category` | reference → `category` | ✓ | カテゴリ（単一・§3.3） |
| `tags` | reference[] → `tag` | – | タグ（複数・任意） |
| `featured` | boolean | – | 注目記事フラグ。トップ/一覧の特集枠で使用 |
| `noIndex` | boolean | – | `true` で `<meta name="robots" content="noindex">`（§6） |
| `showToc` | boolean | – | `true` で目次（ToC）を表示（§4.1） |
| `relatedArticles` | reference[] → `article` | – | 手動指定の関連記事。空なら同カテゴリ等で自動補完（§5） |

### 2.1 配信APIレスポンス例（抜粋）

```jsonc
{
  "title": "なぜ役員は動かないのか？ABMの勝率を変える「刺さるコンテンツ」設計のすべて",
  "slug": "abm-content-design-for-executives",
  "description": "なぜ、あなたのABMはリスト作りで止まってしまうのか？…",
  "content": "<p>…</p><h2 id=\"h01dc09339a\">…</h2>…",
  "featuredImage": { "url": "https://images.microcms-assets.io/.../...png", "width": 1200, "height": 630 },
  "publishedAt": "2026-06-03T00:20:04.065Z",
  "revisedAt":   "2026-06-05T00:58:47.450Z",
  "author":     { "name": "SalesRadar編集部", "slug": "sales-radar", "...": "§3.1" },
  "supervisor": { "name": "猪俣詢", "slug": "makoto_inomata", "role": "SalesRadarユニットマネージャー", "...": "§3.2" },
  "category":   { "name": "データ活用", "slug": "data-utilization" },
  "tags": [],
  "featured": true,
  "noIndex": false,
  "showToc": true,
  "relatedArticles": []
}
```

---

## 3. 関連コンテンツモデル

### 3.1 `author`（執筆・発行主体）

「誰が書いた／発行したか」。多くは編集部（組織）。記事内では冒頭バイライン＋末尾カードで表示（`sr-media-author-wire.html`）。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | string | ✓ | 名称。例「SalesRadar編集部」 |
| `slug` | string | ✓ | プロフィールURL `/column/author/{slug}/` |
| `bio` | textArea | ✓ | 紹介文 |
| `avatar` | image | – | アイコン（推奨 400×400 正方形） |
| `twitter` | string | – | X の ID（`@`なし）。`https://x.com/{twitter}` |
| `type` | select | △ | `organization` / `person`。構造化データの型判定に使用（§7.2）。未設定時は `organization` 既定を推奨 |
| `note` / `linkedin` / `website` 等 | string | – | SNS拡張（`sr-media-author-profile-wire.html` 参照） |

> 例の `author` は編集部＝**Organization**（`twitter: "sales_radar"` あり）。

### 3.2 `supervisor`（監修者）

「内容を専門家が監修したか」。E-E-A-T の信頼性シグナルの中核。記事末尾の監修者カードに表示。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | string | ✓ | 氏名。例「猪俣詢」 |
| `slug` | string | ✓ | プロフィールURL `/column/author/{slug}/`（著者ページと共通テンプレ可） |
| `role` | string | – | 役職・肩書き。例「SalesRadarユニットマネージャー」。**個人監修者で表示** |
| `bio` | textArea | – | プロフィール |
| `avatar` | image | – | 顔写真（推奨 400×400） |
| `twitter` / `note` / `linkedin` / `website` 等 | string | – | SNS拡張（任意。現状スキーマ未定義。追加推奨） |

> 例の `supervisor` は個人＝**Person**（`role` あり）。`author`=組織 と `supervisor`=個人 で、前回設計の Person/Organization 出し分けがそのまま対応する。

### 3.3 `category` / `tag`

| モデル | フィールド | 説明 |
|---|---|---|
| `category` | `name` / `slug` | カテゴリ。例「データ活用」/`data-utilization`。一覧URL `/column/category/{slug}/`。記事は**単一**カテゴリ |
| `tag` | `name` / `slug` | タグ。複数付与可。一覧URL `/column/tag/{slug}/` |

---

## 4. 本文（`content`）の扱い

`content` は microCMS リッチエディタが出力する HTML 文字列。フロントは原則そのまま描画しつつ、以下を後処理する。

### 4.1 目次（ToC）— `showToc`

- `content` 内の `<h2 id="...">` `<h3 id="...">` には microCMS が**自動採番のID**（例 `id="h01dc09339a"`）を付与済み。
- `showToc: true` の場合のみ、本文をパースして h2/h3 を抽出し、記事冒頭またはサイドバーに目次を生成。
- 目次リンクは見出しIDへのアンカー（`#h01dc09339a`）。スクロール追従（現在地ハイライト）はフロント実装。
- `showToc: false` の記事では目次を出さない。

### 4.2 画像

- 本文画像は `<figure><img src="…microcms-assets.io…" alt="…" width="W" height="H"></figure>` 形式。
- `width/height` 属性が入っているため**CLS対策はそのまま流用可**。ただし元画像が大きい（例 2816×1536, 4096×2232）ため、`loading="lazy"` 付与と microCMS 画像API（`?w=&fm=webp&q=`）でのリサイズ配信を後処理で適用推奨。
- `alt` は記事入力時に必ず設定（画像SEO・アクセシビリティ）。

### 4.3 内部リンク（関連記事）

- 本文中に `<a href="https://radar.futurewoods.co.jp/column/{slug}">関連記事：…</a>` のような内部リンクが含まれる。
- フルURL（絶対パス）で入っている点に注意。**自ドメインは相対パス化**するか、`target` を付けずに同タブ遷移とする（外部リンクのみ `target="_blank" rel="noopener"`）。

### 4.4 表（table）

- `<table>` がそのまま出力される。レスポンシブで横溢れするため、`overflow-x:auto` のラッパーを CSS で当てる。

### 4.5 サニタイズ

- CMS入力なので基本信頼できるが、XSS防止のため許可タグ・属性のホワイトリストでサニタイズしてから描画することを推奨。

---

## 5. 関連記事（`relatedArticles`）

- `relatedArticles` が**手動指定**されていればそれを優先表示。
- 空配列（例のケース）の場合は**自動補完**：同 `category` の新着、または同 `tag` を持つ記事から、自身を除外して N件（例3〜4件）を出す。
- 本文内に手書きの関連リンク（§4.3）がある場合と二重にならないよう、表示位置で役割分担（本文中＝文脈リンク／末尾ブロック＝回遊）。

---

## 6. メタ情報・インデックス制御

| 出力 | ソース | 備考 |
|---|---|---|
| `<title>` | `title`（＋サイト名サフィックス） | |
| `<meta name="description">` | `description` | |
| `<link rel="canonical">` | `https://radar.futurewoods.co.jp/column/{slug}/` | 正規URL |
| `<meta name="robots">` | `noIndex` | `true`→`noindex,follow`。既定は索引許可 |
| OGP `og:title` / `og:description` | `title` / `description` | |
| OGP `og:image` | `featuredImage.url`（1200×630） | `og:image:width/height` も付与 |
| `og:type` | `article` | |
| `article:published_time` | `publishedAt` | |
| `article:modified_time` | `revisedAt` | |
| `twitter:card` | `summary_large_image` | |

---

## 7. 構造化データ（JSON-LD）

E-E-A-T と AI 検索（GEO）対応のため、記事ページに以下を出力。

### 7.1 `Article`（本体）

```jsonc
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "{title}",
  "description": "{description}",
  "image": ["{featuredImage.url}"],
  "datePublished": "{publishedAt}",
  "dateModified": "{revisedAt}",
  "mainEntityOfPage": "https://radar.futurewoods.co.jp/column/{slug}/",
  "articleSection": "{category.name}",
  "author": { /* §7.2 */ },
  "publisher": {
    "@type": "Organization",
    "name": "株式会社FUTUREWOODS",
    "logo": { "@type": "ImageObject", "url": "https://.../logo.png" }
  }
}
```

### 7.2 `author` / `supervisor` の型出し分け

`author.type` で `Person` / `Organization` を分岐。`supervisor` は個人想定で `Person`、`reviewedBy` として併記すると監修シグナルが明確になる。

```jsonc
// author（執筆・発行主体）
"author": {
  "@type": author.type === "person" ? "Person" : "Organization",
  "name": author.name,
  "url": "https://radar.futurewoods.co.jp/column/author/" + author.slug + "/",
  // Organization は logo、Person は image / jobTitle
  ...(author.type === "person"
      ? { "image": author.avatar?.url, "jobTitle": author.role }
      : { "logo": author.avatar?.url }),
  "sameAs": [
    author.twitter && "https://x.com/" + author.twitter,
    author.note    && "https://note.com/" + author.note,
    author.linkedin, author.website
  ].filter(Boolean)
}

// supervisor（監修者）— 存在する場合のみ
"reviewedBy": {
  "@type": "Person",
  "name": supervisor.name,
  "jobTitle": supervisor.role,
  "image": supervisor.avatar?.url,
  "url": "https://radar.futurewoods.co.jp/column/author/" + supervisor.slug + "/"
}
```

### 7.3 `BreadcrumbList`

`ホーム > コラム > {category.name} > {title}` をパンくず構造化。

### 7.4 `FAQPage`（任意・該当記事のみ）

本文に Q&A セクション（例：「Q1. …」「A. …」）がある記事は、`FAQPage` を併設するとリッチリザルト/AI引用に有利。Q&A の有無は本文パターン検出か、CMSに `faq`（質問・回答の繰り返し）フィールドを追加して構造的に持つのが確実（本文HTMLからの抽出は壊れやすいため後者推奨）。

```jsonc
{
  "@type": "FAQPage",
  "mainEntity": [
    { "@type": "Question", "name": "Q1…",
      "acceptedAnswer": { "@type": "Answer", "text": "A…" } }
  ]
}
```

---

## 8. ページ構成（記事詳細 `/column/{slug}/`）

| 位置 | 要素 | ソース |
|---|---|---|
| パンくず | ホーム > コラム > カテゴリ > タイトル | `category` / `title` |
| 記事ヘッダ | カテゴリ・タイトル・公開/更新日・**冒頭バイライン（著者）** | `category`/`title`/`publishedAt`/`revisedAt`/`author` |
| アイキャッチ | `featuredImage`（1200×630） | `featuredImage` |
| 目次 | ToC（`showToc=true` 時） | `content` の h2/h3 |
| 本文 | リッチHTML（画像・表・内部リンク・FAQ） | `content` |
| CTA | 無料デモ・資料DL（本文中＋末尾） | 固定 |
| **監修者カード** | 末尾の監修者プロフィール（`supervisor` がある場合） | `supervisor` |
| 著者カード | 末尾の執筆者プロフィール | `author` |
| 関連記事 | 手動 or 自動補完 | `relatedArticles` / `category` |

> 監修者・著者カードのワイヤーは `sr-media-author-wire.html`、プロフィールページは `sr-media-author-profile-wire.html` を参照。

---

## 9. 補足・運用注意

- **著者（author）と監修者（supervisor）は別物**。執筆＝編集部（組織）、監修＝個人専門家、という役割で運用する。両方表示する場合は「執筆：SalesRadar編集部／監修：猪俣詢」のように並べる。
- `slug` は英数ハイフン（例 `abm-content-design-for-executives`）。日本語・連番IDより可読性・SEOで有利。一度公開した `slug` は変更しない（変更時は 301 リダイレクト必須）。
- `featuredImage` は OGP 兼用のため **1200×630 を厳守**（SNSシェア時の見栄え）。
- `description` 未入力時は本文先頭からの自動抽出をフォールバックにするが、原則手動設定。
- `noIndex` は下書き・薄い記事・重複対策に使用。サイトマップ（`sitemap.xml`）からも除外する。
- `supervisor` に SNS（`twitter`/`note` 等）を表示したい場合は §3.2 のとおりスキーマ拡張が必要（現状未定義）。
