# 引き継ぎメモ（proposal-line）

## 最終更新
2026-10-13 新規作成：schedule-line をベースに proposal-line を作成（授業内容提案書の配信）

## プロジェクト概要
- 春日部アカデミーの「授業内容提案書（受講内容確認書）」JPGを、生徒・保護者へ **LINE配信** するツール
- **夏／冬／春**に実施（講習前に提案書を配る運用）
- 姉妹プロジェクト **`schedule-line`**（授業日程表の配信）を**コピーして作成**。
  **配信ロジックは schedule-line と完全に同一**で、異なるのは
  「送る画像の中身・画面文言・画像フォルダ」だけ。
- Google Apps Script（GAS）+ clasp 運用。管理画面はGASのWebアプリ
- 送信先は schedule-line と**同じ共有スプレッドシート `line-recipients` を共用**（新規作成不要）

## システム構成

```
[proposal-line-images フォルダ(Drive)]   生徒ごとの提案書JPG（ファイル名＝生徒氏名）
                                         ＋ 全員共通画像（「_」始まり。例 _common1.jpg）
            │
            ▼
[proposal-line GAS (Webアプリ)]
   ├─ doGet ?image=氏名     … 画像のブラウザ確認用プレビュー
   ├─ 管理画面(admin.html)  … あいさつ文入力／プレビュー／テスト・本配信／削除
   ├─ getDeliveryTargets    … 画像と line-recipients を照合
   ├─ testSend/execute…     … LINE Messaging API で配信
   └─ deleteDeliveredImages … 配信後にJPGをゴミ箱へ
            │                         ▲
            ▼                         │
[共有スプレッドシート line-recipients]   │  ※ schedule-line と共用
   ├─ Families シート（保護者ID・配信有効フラグ）
   └─ Students シート（生徒氏名・生徒ID・在籍）─┘  ※家族IDで結合
            │
            ▼
[LINE Messaging API push] → 生徒・保護者のLINEへ「あいさつ文＋個別画像＋共通画像」
```

### 主な関数
| 機能 | 主な関数 |
|---|---|
| 画像配信の仕組み | `_serveImage` / `_buildImageUrls` |
| 共有シート照合・管理画面 | `getDeliveryTargets` / `_computeDelivery` |
| テスト配信・本配信 | `testSendSchedule` / `executeScheduleDelivery` / `_pushLineMessages` |
| 配信後の画像削除 | `deleteDeliveredImages` |

## schedule-line からの変更点（これだけ）
- 画面タイトル：`proposal-line 授業内容提案書配信`
- サブタイトル：`春日部アカデミー ／ 授業内容提案書JPGを生徒・保護者へLINE配信`
- あいさつ文プレースホルダ：提案書向け文面
- 配信先ラジオの既定：`保護者＋生徒 両方`（毎回生徒にも見せる運用。※元々の既定も both）
- **画像フォルダID**：`proposal-line-images`（`1RnVJA2cSyAypO7UZsySNU5mY3TfB4Cf1`）
  - proposal-line では**コード側の定数 `IMAGE_FOLDER_ID` に固定**（`コード.js`）。
    schedule-line と値が異なるため、プロパティではなくコードで持つ。
    ふくちさんが入力するプロパティは LINEトークンと `RECIPIENTS_SPREADSHEET_ID` の2つのみ。

## 実運用フロー（夏／冬／春に1回）

1. **生徒ごとの提案書JPGを作成**
   - ファイル名＝生徒氏名（例：`古内伶奈.jpg`）
   - **氏名にスペースを入れない**（照合はスペース無視だが、揃えておくと安心）
   - 拡張子は `.jpg` / `.jpeg`（大文字小文字は不問）
   - 全員共通の画像（0枚〜複数枚）はファイル名を `_` 始まりにする（例：`_common1.jpg`）
     - 個別照合の対象外。ファイル名の自然順（_common2 → _common10）で送る
2. **`proposal-line-images` フォルダに置く**
   - フォルダは「リンクを知っている全員が閲覧可」に共有済みであること（配信URLが公開画像を返すため）
3. **管理画面を開く**（本番URL＝デプロイした `/exec`）
4. **あいさつ文を入力**（テキストエリア。改行可）
   - **配信先を選択**（保護者のみ／生徒のみ／保護者＋生徒 両方。既定は両方）
5. **「配信対象プレビュー」で確認**
   - 配信対象◯名／送信予定◯通／警告◯件
   - 警告（対応表に氏名なし／同名複数／在籍FALSE／配信有効FALSE／送信先ID未登録）が無いか確認
   - 生徒LINE未所持の子は「生徒ID未登録→生徒はスキップ」と表示（保護者には届く）
   - 「この生徒に届く通数」列は per-生徒 の指標。兄弟は保護者宛て1通を共有するため、
     列の合計は「送信予定」より多くなることがある（注記あり）。実際に送る回数は送信予定の通数
6. **「🧪 テスト配信」で見え方確認**（選択した配信先での送信リスト先頭1件分が、分割も含めそのままふくちさんのLINEにのみ届く）
7. **「🚀 本配信」**（全生徒＋保護者へ）
   - 事前確認ダイアログ → `confirm` → `prompt` に「配信」と入力、の多重確認
8. **配信成功後、「📁 配信済み画像を削除する」ボタンで画像をゴミ箱へ**
   - このボタンは本配信成功後にのみ表示される（うっかり削除防止）
   - ゴミ箱移動なので、必要なら復元可能

### 配信の通数ルール
- 各生徒の画像を、選択した配信先（生徒本人ID／保護者ID／両方）のうち空でないIDへ送る
- 同一 LINE_USER_ID には1回の配信だけ（重複除去）
  - **兄弟で保護者IDが同じ場合は1回にまとめる**：あいさつ文 → 兄弟の個別画像（兄弟順位順）→ 共通画像
- 1回の配信の中身：あいさつ文 → 個別画像 → 共通画像（ファイル名の自然順）
  - LINEは1 pushあたり最大5メッセージ。超える場合は複数pushに分割（あいさつ文は最初のpushのみ）
  - 途中のpushが失敗したら、その宛先の残りpushは送らない（順序崩れ防止）。エラー詳細に何回目か表示
- 除外：在籍フラグFALSE（生徒単位）／配信有効フラグFALSE（Families。家族全員）／同名の生徒が複数（判別不能）

## 環境情報

### リポジトリ
- URL：https://github.com/k-acdm/proposal-line
- Visibility：（作成済みリポジトリに準ずる）
- ブランチ：dev / main（dev で作業 → main にマージ）

### GASプロジェクト
- 名前：proposal-line
- スクリプトID：`1q8QyOBA7Kdosmr8BWIUWP_ccD7YcKafJLEgpBcX4CEvcttFqfaY94MzI`
- エディタURL：https://script.google.com/d/1q8QyOBA7Kdosmr8BWIUWP_ccD7YcKafJLEgpBcX4CEvcttFqfaY94MzI/edit
- 本番URL：初回デプロイ後に発行（ふくちさんがデプロイ）
- clasp ログインアカウント：kasukabe.academy@gmail.com
- ⚠️ **schedule-line のGASプロジェクト（`1XahVQ…`）とは別物**。proposal-line の `src/gas/.clasp.json`
  には上記 `1q8QyOBA…` が入っている。schedule-line には絶対に push しないこと。

### clasp 構成
- `.clasp.json` は `src/gas/` に配置（rootDir="" = src/gas がソースルート）
- clasp コマンドは `~/proposal-line/src/gas` で実行する
- ソースファイル：`コード.js`（サーバー）/ `admin.html`（管理画面）/ `appsscript.json`

### 共有スプレッドシート（line-recipients）※schedule-line と共用・照合対象
- 列は**1行目のヘッダ名**で特定（列順が変わっても可。ヘッダ名は変えないこと）
- **Families シート**：家族ID | 宛名 | 配信区分 | 保護者LINE_USER_ID | 登録日 | 配信有効フラグ
  - 配信有効フラグ FALSE（空欄も FALSE 扱い）→ その家族は生徒・保護者とも配信しない
- **Students シート**：生徒ID | 家族ID | 学年 | 生徒氏名 | 兄弟順位 | 在籍フラグ | 生徒LINE_USER_ID
  - 生徒LINE_USER_ID は空欄可（LINE未所持）
- フラグは `TRUE`/`1`/`○`/`はい`/`在籍` を真とみなす

### スクリプトプロパティ一覧
ふくちさんがGASエディタ（プロジェクトの設定 → スクリプト プロパティ）で入力する。
| キー | 必須 | 入れる値 |
|---|---|---|
| `LINE_MESSAGING_CHANNEL_ACCESS_TOKEN` | ✅必須 | LINE送信用トークン（**schedule-line と同じ値**） |
| `RECIPIENTS_SPREADSHEET_ID` | ✅必須 | 共有シート line-recipients のID（**schedule-line と同じ値**） |
| `ADMIN_LINE_USER_ID` | テスト配信を使う場合に必要 | ふくちさんのLINE_USER_ID（schedule-line と同じ値） |
| `IMAGE_URL_METHOD` | 任意 | `lh3`（既定）/ `thumbnail`。LINEで画像が出ないとき `thumbnail` |

- **`IMAGE_FOLDER_ID` はプロパティ不要**：コード側の定数で `proposal-line-images`（`1RnVJA2cSyAypO7UZsySNU5mY3TfB4Cf1`）に固定済み。
- `WEB_APP_URL` / `SPREADSHEET_ID` は proposal-line では不要（コードから参照しない）。

### oauthScopes（appsscript.json）
- `script.external_request`（LINE API 送信）
- `drive`（フォルダ読み取り＋配信後のJPG（共通画像含む）ゴミ箱移動）
- `spreadsheets`（line-recipients 読み取り）

## 重要な学び（schedule-line から引き継ぎ）

1. **GASエディタは編集後F5リロードしないと旧コードを掴む**（デプロイ前に必ずリロード）
2. **`clasp push` 成功表示でもリモートHEADが旧版に戻ることがある**
   - 疑わしいときは一時フォルダへ `clasp pull` して中身を検証する
     （別ディレクトリに `.clasp.json` をコピー→`clasp pull`→grepで関数の有無を確認）
3. **スコープ変更後は再認証が必要**
   - 再認証ダイアログが出ない場合は Google権限設定（https://myaccount.google.com/permissions）で
     proposal-line のアクセスを一旦削除 → 関数を再実行して再認証を強制
4. **画像URLの使い分け**
   - `?image=氏名`：GASが返すのは **HTMLプレビュー**（ブラウザ確認用）。**LINEには使えない**
   - LINE配信用URL：`_buildImageUrls(fileId)` の **lh3 / thumbnail 直リンク**（本物のJPEG）
   - `IMAGE_URL_METHOD` で `lh3` ⇄ `thumbnail` を切替（再デプロイ不要）
   - 前提：画像フォルダが「リンクを知っている全員が閲覧可」で共有されていること
5. **デプロイは「既存デプロイの編集 → 新バージョン」**（新規デプロイはURLが変わるので禁止）
6. **Webアプリ設定「アクセスできるユーザー：全員」は必須**（`webapp.access: "ANYONE_ANONYMOUS"` を維持）
7. **clasp 3.x の `clasp create` は appsscript.json を既定版で上書きする**
   - proposal-line 作成時に発生。作成後に Asia/Tokyo・ANYONE_ANONYMOUS・oauthScopes を復元した

## 運用ルール（billing-line / schedule-line と共通）

### ⚠️ git add の注意
- `git add .` は必ず **`~/proposal-line` ルート**で実行する
- `src/gas` 配下で実行すると docs/HANDOFF.md が漏れる

### dev/main 切替
- **再開時**：`git checkout dev` → `git pull origin dev`
- **終了時**：`git pull origin dev` → `git checkout main` → `git merge dev` → `git push origin main` → `git checkout dev`

### 本番反映
1. `git pull origin dev`
2. `git checkout main` → `git merge dev` → `git push origin main`
3. `clasp push`（GAS変更時。`~/proposal-line/src/gas` で実行）
4. GASエディタをF5リロード →「デプロイを管理」→ 既存デプロイを編集 →「新バージョン」→ デプロイ
