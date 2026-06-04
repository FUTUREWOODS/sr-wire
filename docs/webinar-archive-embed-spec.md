# SalesRadar ウェビナー アーカイブ｜YouTube埋め込み仕様

関連ワイヤーフレーム：

- 一覧・登録：`sr-webinar-wire.html`
- 視聴ページ：`sr-webinar-archive-wire.html`

---

## 1. 概要

過去開催ウェビナーのアーカイブを、**YouTube（限定公開）動画の埋め込み**で配信する。リード獲得フォーム登録後、SendGrid から送付される視聴ページURLに着地し、ページ内の YouTube プレーヤーで視聴する。

- 配信基盤：YouTube（限定公開 / Unlisted）
- ゲート：リード獲得フォーム（**ソフトゲート**。アクセス制御ではなくリード獲得が目的）
- メール送付：SendGrid（Transactional Email）

---

## 2. 全体フロー

```
[一覧ページ] アーカイブカード「視聴する」
      ↓ どのアーカイブか(archive_id)を引き継ぎ
[アーカイブ視聴フォーム] 会社名・氏名・メール＋同意（必須）
      ↓ 送信（フォーム → サーバー/MA）
[サーバー] リード保存（CRM/MA）＋ archive_id と紐付け
      ↓ SendGrid API 呼び出し
[SendGrid] 視聴ページURLを記載したメールを自動送付
      ↓ メール内リンク
[視聴ページ /webinar/archive/{slug}/] YouTube限定公開を iframe 埋め込み
```

- **2回目以降**：Cookie 等で登録済みと判定し、フォームを出さずワンクリックで視聴ページへ。視聴アクション（archive_id）のログのみ記録。
- 「開催のお知らせを受け取る」opt-in は SendGrid の通知用コンタクトリストへ登録（将来セミナーの一斉配信用）。

---

## 3. YouTube 側の設定

| 項目 | 設定 | 理由 |
|---|---|---|
| 公開設定 | **限定公開（Unlisted）** | 「非公開」は埋め込み不可。「公開」は検索に露出してしまう |
| 埋め込み | 許可（ON） | iframe 表示に必須 |
| 関連動画 | 自社チャンネルに限定（`rel=0`） | 他社動画への離脱を抑制 |
| 終了画面 / カード | 自社の他アーカイブ・デモ誘導に設定 | 視聴後の送客 |

> ⚠️ 限定公開URL・動画IDを知っていればフォームを通さず視聴可能。厳密な視聴制限が必要な場合は別基盤（Vimeo のドメイン制限、署名付きURL配信等）を検討。

---

## 4. 埋め込み iframe 仕様

### 基本形

```html
<iframe
  src="https://www.youtube-nocookie.com/embed/{VIDEO_ID}?rel=0&modestbranding=1"
  title="{ウェビナータイトル}"
  loading="lazy"
  allow="accelerated-2d-canvas; autoplay; encrypted-media; picture-in-picture"
  allowfullscreen
  referrerpolicy="strict-origin-when-cross-origin"
  style="width:100%; aspect-ratio:16/9; border:0; border-radius:8px;">
</iframe>
```

### パラメータ

| パラメータ | 値 | 内容 |
|---|---|---|
| `rel` | `0` | 関連動画を同チャンネルに限定 |
| `modestbranding` | `1` | YouTubeロゴを最小化 |
| `autoplay` | `0`（既定） | 自動再生はしない（UX・規約配慮） |
| `start` | 任意 | 開始位置（秒）指定が必要な場合のみ |

### 推奨事項

- ドメインは `youtube-nocookie.com`（プライバシー強化モード）を使用。
- レスポンシブは `aspect-ratio: 16/9` で対応（ワイヤーの `.player` と同じ）。
- `loading="lazy"` でファーストビュー外の遅延読み込み。

---

## 5. 視聴ページ仕様

- URLパス：`/webinar/archive/{slug}/`
  - `{slug}` 例：`2026-04-target-design`（連番IDより可読性・SEOで有利）
- `<meta name="robots" content="noindex, nofollow">`（検索露出させない）
- 構成要素（`sr-webinar-archive-wire.html` 参照）：
  - YouTube 埋め込みプレーヤー（16:9）
  - タイトル / 開催日 / 登壇者 / 収録時間
  - 概要・おすすめ対象・アジェンダ（タイムスタンプ）
  - 投影資料ダウンロード（登録済みのため追加入力なし）
  - 関連アーカイブ（サイドバー）
  - デモ・資料請求への CTA

---

## 6. サムネイル

一覧カード（`sr-webinar-wire.html`）と関連アーカイブのサムネは YouTube サムネイルを流用可。

```
https://img.youtube.com/vi/{VIDEO_ID}/hqdefault.jpg   （480x360）
https://img.youtube.com/vi/{VIDEO_ID}/maxresdefault.jpg（高解像度・存在しない場合あり）
```

---

## 7. データ項目

### アーカイブ視聴フォーム

| 項目 | 必須 | 備考 |
|---|---|---|
| 視聴したいアーカイブ | ✓ | プルダウン。カード遷移時は `archive_id` を hidden field で自動セット |
| 会社名 | ✓ | |
| お名前 | ✓ | |
| メールアドレス | ✓ | 視聴ページURLの送付先 |
| 部署・役職 | – | |
| 電話番号 | – | |
| 今後の開催のお知らせを受け取る | – | 任意・初期ON。通知リストへ登録 |
| プライバシーポリシー同意 | ✓ | |

### ウェビナー／アーカイブのコンテンツ定義

ウェビナー・アーカイブ・登壇者の各コンテンツは **CMS（ヘッドレスCMS）で管理**する。スキーマ・公開ステータス・API取得方針は §8 を参照。

---

## 8. CMS によるコンテンツ管理とスキーマ

### 8.1 方針

- ウェビナー（開催予定／アーカイブ）・登壇者の情報は、コードに直書きせず **ヘッドレスCMS** で管理する（例：microCMS / Contentful 等）。
- フロント（一覧ページ `/webinar/`、視聴ページ `/webinar/archive/{slug}/`）は CMS の **配信API**からコンテンツを取得して描画（SSG/ISR 推奨）。
- **1つの `webinar` モデルで開催予定とアーカイブの両方を表現**し、`status` で出し分ける。開催が終わったエントリに `youtube_video_id` 等のアーカイブ項目を追加すると、自動的にアーカイブ一覧・視聴ページへ反映される。
  - （別案：`webinar` と `archive` を別モデルに分離。運用がシンプルだが終了→アーカイブ化の手当が二重管理になりやすいので非推奨）

### 8.2 コンテンツモデル：`webinar`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `slug` | string | ✓ | URL識別子。例 `2026-04-target-design`。`/webinar/archive/{slug}/` に使用 |
| `title` | string | ✓ | ウェビナータイトル |
| `status` | select | ✓ | `coming_soon`（受付前）／`open`（受付中）／`live`（配信中）／`archived`（アーカイブ） |
| `event_date` | datetime | ✓ | 開催日時 |
| `duration_min` | number | – | 収録/開催時間（分） |
| `speakers` | reference[] | – | `speaker` モデルへの参照（複数可） |
| `summary` | richText | – | 概要 |
| `target_audience` | richText / list | – | こんな方におすすめ |
| `agenda` | repeater | – | アジェンダ（`time` + `label` の繰り返し） |
| `thumbnail` | image | – | サムネイル（未設定時は YouTube サムネを流用） |
| `apply_form_url` | string | – | 開催予定の専用申込フォームURL（`status` が `archived` 以外で使用） |
| `capacity` | number | – | 定員 |
| **アーカイブ用（`status=archived` 時に設定）** | | | |
| `youtube_video_id` | string | △ | 限定公開動画ID。`archived` では必須 |
| `material_url` | file/string | – | 投影資料（PDF）URL |
| `published_at` | datetime | – | アーカイブ公開日 |

> `status` が `archived` のエントリ → アーカイブ一覧＋視聴ページに表示。
> それ以外 → 開催予定リストに表示（`apply_form_url` の専用フォームへ誘導）。

### 8.3 コンテンツモデル：`speaker`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | string | ✓ | 氏名 |
| `company` | string | – | 所属・会社名 |
| `title` | string | – | 役職 |
| `avatar` | image | – | 顔写真 |
| `profile` | richText | – | プロフィール |

### 8.4 配信APIレスポンス例（`webinar` / アーカイブ）

```json
{
  "slug": "2026-04-target-design",
  "title": "優良顧客の勝ちパターンを可視化する",
  "status": "archived",
  "event_date": "2026-04-15T13:00:00+09:00",
  "duration_min": 60,
  "speakers": [
    { "name": "山田 太郎", "company": "FUTUREWOODS株式会社", "title": "セールスマネージャー" }
  ],
  "summary": "SalesRadarを使ったターゲット選定の考え方を、デモを交えて解説します。",
  "agenda": [
    { "time": "00:00", "label": "イントロダクション" },
    { "time": "05:00", "label": "類似企業分析の考え方" },
    { "time": "20:00", "label": "SalesRadar 操作デモ" },
    { "time": "45:00", "label": "Q&A" }
  ],
  "youtube_video_id": "xxxxxxxxxxx",
  "material_url": "https://.../target-design.pdf",
  "published_at": "2026-04-20T10:00:00+09:00"
}
```

### 8.5 CMSとフォーム／配信の責務分担

| 対象 | 管理場所 | 備考 |
|---|---|---|
| ウェビナー・アーカイブ・登壇者 | **CMS** | 本仕様のスキーマ |
| 視聴フォームの送信・リード保存 | フォーム/MA・CRM | CMSの `slug`/`archive_id` を紐付け |
| 視聴URL・通知メール送付 | SendGrid | §9 |
| 動画本体 | YouTube（限定公開） | CMSは `youtube_video_id` のみ保持 |

---

## 9. SendGrid 連携

- 用途：**Transactional Email**（視聴URL自動送付）／開催通知の一斉配信。
- 視聴URL送付は Dynamic Template を使用し、差し込み変数で出し分け。

### テンプレート差し込み変数（例）

| 変数 | 内容 |
|---|---|
| `{{first_name}}` | 宛名 |
| `{{webinar_title}}` | アーカイブタイトル |
| `{{view_url}}` | 視聴ページURL（`/webinar/archive/{slug}/`） |
| `{{material_url}}` | 資料DL URL（任意） |

### 注意

- 視聴URLは「視聴ページ」を指す（YouTube URLを直接送らない＝動画ID露出・拡散の抑制）。
- 拡散対策を強める場合は、視聴ページURLにワンタイムトークンを付与しサーバー側で検証（→ §10）。
- opt-in したコンタクトは通知用リストへ追加し、配信停止（Unsubscribe）リンクを必ず設置。

---

## 10. ワンタイムトークン方式（オプション）

視聴ページURLの共有による無制限アクセスを抑止したい場合のオプション。SendGrid のメールに静的URLではなく**固有トークン付きURL**を載せ、サーバー側で検証してから視聴ページを表示する。

> ⚠️ **限界**：トークンは「ページURLの共有」を防ぐもの。ページ表示後の `iframe src` には YouTube動画IDが残るため、DOM から動画IDを抽出すれば限定公開動画自体は視聴可能。動画そのものを保護したい場合はトークンでは不十分で、Vimeo（ドメイン/視聴制限）や署名付き配信が必要。

### 10.1 フロー

```
[フォーム送信 / 登録済みユーザーの視聴要求]
   ↓ サーバーがトークン発行（lead_id + archive_id + 有効期限）
[SendGrid] view_url = https://radar.../webinar/view?t={token} を送付
   ↓ メール内リンクをクリック
[サーバー] トークン検証
   ├─ 有効 → 短命セッションCookie発行 → /webinar/archive/{slug}/ へ302（URLからトークン除去）→ 埋め込み表示
   └─ 無効/期限切れ → 再発行ページ（メール再送）へ誘導
```

- トークンは**ページ表示のトリガーにのみ**使用し、表示後は短命Cookieでセッション管理（リロード共有耐性・履歴へのトークン残り対策）。
- 「ワンタイム」は厳密な1回ではなく、**期限内・本人は何度でも視聴可**とするのが現実的（動画は再生・リロード前提）。
- 期限切れは「視聴できない方はこちら」からメール再送で救済。

### 10.2 トークン設計

| 項目 | 推奨 | 補足 |
|---|---|---|
| 中身 | `lead_id` + `archive_id` + `exp` | 誰が・どのアーカイブを・いつまで |
| 方式 | **ステートフル（DB保存）** 推奨 | 失効・利用回数制御・監査が可能。軽量にするならJWT署名のステートレスも可（個別失効不可） |
| 有効期限 | 7〜30日 | メールクリックまでの猶予 |
| 利用回数 | `max_views`（任意・null=無制限） | 不正共有検知の閾値としても利用可 |
| 再発行 | 期限切れ時にメール再送で新トークン発行 | 旧トークンは失効 |

### 10.3 URL例

- メール内（トークン付き）：`/webinar/view?t=<token>`（または `/webinar/v/{token}`）
- 検証後の表示URL（クリーン）：`/webinar/archive/{slug}/`

### 10.4 テーブルスキーマ（ステートフルの場合）

トークンと視聴ログを保持する。CMS（§8）とは別に、アプリ側DBで管理する。

```sql
-- 視聴トークン
CREATE TABLE archive_view_token (
  id            BIGINT      PRIMARY KEY AUTO_INCREMENT,
  token         CHAR(43)    NOT NULL UNIQUE,      -- URLセーフな乱数（例: 256bit base64url）。本体はハッシュ保存を推奨
  lead_id       BIGINT      NOT NULL,             -- リード（フォーム登録者）への参照
  archive_slug  VARCHAR(120) NOT NULL,            -- CMSの webinar.slug（archived エントリ）
  email         VARCHAR(255) NOT NULL,            -- 送付先・本人判定用
  max_views     INT         NULL,                 -- 上限（NULL=無制限）
  view_count    INT         NOT NULL DEFAULT 0,   -- 利用回数
  expires_at    DATETIME    NOT NULL,             -- 有効期限
  revoked_at    DATETIME    NULL,                 -- 失効（再発行・手動失効時にセット）
  created_at    DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  last_used_at  DATETIME    NULL,
  INDEX idx_token (token),
  INDEX idx_lead_archive (lead_id, archive_slug),
  INDEX idx_expires (expires_at)
);

-- 視聴ログ（任意・興味シグナル／不正検知用）
CREATE TABLE archive_view_log (
  id            BIGINT      PRIMARY KEY AUTO_INCREMENT,
  token_id      BIGINT      NULL,                 -- archive_view_token.id への参照（NULLはCookieセッション経由）
  lead_id       BIGINT      NOT NULL,
  archive_slug  VARCHAR(120) NOT NULL,
  viewed_at     DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  ip            VARBINARY(16) NULL,               -- 不正共有検知用（保持はプライバシーポリシーに準拠）
  user_agent    VARCHAR(255) NULL,
  INDEX idx_lead (lead_id),
  INDEX idx_archive (archive_slug),
  INDEX idx_viewed (viewed_at)
);
```

- `token` 列は生トークンではなく**ハッシュ（SHA-256等）で保存**し、URLには生トークンを使う（漏洩時の被害低減）。
- `archive_view_log` は「どのアーカイブを誰が視聴したか」の興味シグナルとして CRM/MA 連携にも活用できる（§7 のアーカイブ単位の視聴記録と同義）。
- ステートレス（JWT）採用時はこれらのテーブルは不要だが、`max_views`・個別失効・視聴ログは取得できない。

### 10.5 検証の擬似コード

```
GET /webinar/view?t=TOKEN
  rec = token.find(hash(TOKEN))
  if !rec || rec.revoked_at || now > rec.expires_at:           → 再発行ページ
  if rec.max_views != null && rec.view_count >= rec.max_views: → 再発行ページ
  rec.view_count++ ; rec.last_used_at = now
  log.insert(archive_view_log, rec.lead_id, rec.archive_slug)
  session.set('archive_access:'+rec.archive_slug, true, ttl=2h)  -- 短命Cookie
  302 → /webinar/archive/{rec.archive_slug}/                      -- URLからトークン除去

GET /webinar/archive/{slug}/
  if !session.has('archive_access:'+slug):  → 視聴フォームへ
  render(YouTube embed)                       -- 動画IDはDOMに出る（前述の限界）
```

### 10.6 SendGrid／CMSとの関係

- CMS（§8）：`youtube_video_id` 等のコンテンツのみ保持（変更なし）。
- トークン発行・検証・視聴ログ：アプリ側サーバーが担当（上記テーブル）。
- SendGrid（§9）：Dynamic Template の `{{view_url}}` に**トークン付きURL**を差し込む（静的URLから置き換え）。

---

## 11. 補足・注意点まとめ

- ゲートはソフトゲート（限定公開URLは知っていれば視聴可）。リード獲得目的と割り切る。
- さらに拡散を抑えたい場合はワンタイムトークン方式（§10）を採用。ただし動画ID自体の露出は残るため、完全保護には別配信基盤が必要。
- 視聴ページは `noindex`、YouTube動画は限定公開で、検索流入は一覧ページ（`/webinar/`）に集約する。
