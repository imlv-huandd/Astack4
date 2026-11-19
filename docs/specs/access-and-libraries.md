# 認証・プロジェクト・ライブラリ

確認基準と資料の区分は [索引](../README.md) を参照。

## 認証と権限

管理画面はJWT認証を使用し、パスワードはBCryptで処理する。`/api/auth/login`、`/refresh`、`/me`、`/logout` を提供する。HTTPセッションはSTATELESS。Windows認証を使う公開閲覧は別の認証・ACL判定経路を持つ。

管理操作は `ADMIN`、一般利用は `USER` を基準とする。SecurityConfigのURL判定とController・Serviceの判定の両方が適用される。旧仕様の `OPERATOR` 表記だけを根拠に操作を許可してはいけない。例えばスケジュールのメソッドにはOPERATORが残るが、URLの既定管理者制限も適用される。

| 操作 | ADMIN | USER |
| --- | --- | --- |
| プロジェクト閲覧 | 全プロジェクト | 参加プロジェクト |
| プロジェクト作成・変更・削除、参加者管理 | 可 | 不可 |
| ライブラリ作成、ビルド・AI実行 | 可 | 参加プロジェクト内 |
| ライブラリの既存設定編集 | 可 | 不可 |
| ゴミ箱へ移動・復元 | 可 | 参加先で自分が作成したもの |
| 完全削除、プロジェクトのゴミ箱を空にする | 可 | 不可 |
| 接続・エンジン・プロファイル・テンプレート・ビューア・ユーザー管理 | 可 | 不可 |

参加者のない既存プロジェクトは管理者が利用する。既存データに参加者や作成者を推測して補完する処理はない。

## ユーザー管理と初回パスワード変更

新規作成したUSER/ADMINと、新規環境の初期管理者は `passwordChangeRequired=true` で作成する。初期パスワードでの認証成功だけでは解除せず、本人による変更完了まで管理・業務操作を許可しない。作成・更新・削除・操作履歴の参照はADMINのみ。

### 認証と本人による変更

- `POST /api/auth/login` は変更必須の場合、用途 `password_change`・有効期間10分のJWTを発行し、リフレッシュトークンは発行しない。既存リフレッシュトークンも削除する。
- 専用JWTは本人情報取得・パスワード変更・ログアウトに使用できる。業務APIは `403`、`code=PASSWORD_CHANGE_REQUIRED` を返す。JWTの用途と最新のDB状態の両方で制限し、通常のJWTがあってもDB上で変更必須なら制限する。
- 画面初期化では `/api/auth/me` から最新情報を取得し、変更必須なら `/change-password` へ誘導する。再読み込み・直接URL・別タブで回避できない。変更専用画面には業務メニューやスキップを設けず、ログアウトを用意する。
- パスワード変更の入力はジョブ実行・ストレージ接続と共通の `FormModal` を使用する。通常はヘッダーから現在の画面に重ねて開き、キャンセル・閉じるで元の画面と入力済みの絞り込み条件を保持する。必須変更では業務画面を表示せず、閉じる・ログアウトはログアウトとして扱う。送信中は閉じる操作と二重送信を抑止する。
- `POST /api/auth/change-password` は `currentPassword`、`newPassword`、`confirmPassword` を受け付ける。対象ユーザーはJWTの本人とし、現在のパスワード・新旧の相違・確認入力の一致を検証する。
- 新規作成・管理者再設定・本人変更の新しいパスワードは、半角英数字・記号（ASCIIのU+0021〜U+007E）で8〜72文字とする。日本語・全角文字・絵文字・空白・改行・タブ・制御文字は許可しない。英字・数字・記号の全種類を含める条件は設けない。画面とサーバーで共通の条件を適用する。1文字が1バイトとなる文字に限定し、BCryptの72バイト上限に合わせる。
- この入力条件は新規設定・変更時に適用する。ログインと本人変更時の現在のパスワード照合には適用せず、従来の日本語・絵文字・空白を含むパスワードでも認証できる状態を維持する。既存パスワードの一括変更や切り詰めは行わない。
- 成功時はパスワード、変更必須解除、本人変更日時、セッション世代、操作履歴を同一トランザクションで更新し、リフレッシュトークンを削除する。応答は `204`。画面の認証情報を消去し、新パスワードで再ログインする。
- 同時変更ではロック取得後に資格情報を再検証し、一度のみ成功させる。入力不正・更新失敗時は変更必須状態を維持する。
- `/refresh` は最新の有効状態と変更必須状態を確認し、無効または変更必須なら通常JWTを発行しない。更新では最終ログイン日時を変更しない。
- Windows認証による公開閲覧・匿名の公開検索は従来の認証・ACL経路を使用する。

### 項目と画面

| 項目 | 記録・表示 |
| --- | --- |
| 作成日時・作成者 | 既存の `createdAt` / `createdBy`。変更不可。初期管理者の作成者は `system` |
| 更新日時・更新者 | 既存の `updatedAt` / `updatedBy`。管理情報の更新時に記録 |
| 最終ログイン日時 | パスワード認証成功時のみ更新。管理監査項目は変更しない |
| 有効・無効 | 無効時は新規ログイン・既存JWT・更新トークンを拒否 |
| 変更必須 | 新規作成・管理者による他ユーザーのパスワード再設定で必須にする |
| 本人のパスワード変更日時 | 本人が専用APIで変更完了した日時。管理者による再設定では変更しない |

ユーザー管理はジョブ管理と同じ下線付きタブで「ユーザー一覧」と「操作履歴」を切り替える。選択状態は `tab=users` / `tab=history` でURLに保持し、直接URL・再読み込み・ブラウザの戻る操作に対応する。両一覧は共通の `ListTable`、カード・検索ツールバー・`ListPagination` を使い、見出し・罫線・行の余白・空表示・読み込み表示をそろえる。

ユーザー一覧はユーザーID・名前の部分一致、ロール、有効状態、変更必須状態でサーバー側検索・絞り込み・ページングする。作成情報と最終ログイン日時、状態を一覧に表示し、更新情報と本人変更日時は、操作列の「詳細」から開く閲覧用ダイアログに表示する。一覧内で行を展開せず、背景の一覧・検索条件・ページ番号・スクロール位置を保持する。詳細は既存のジョブ詳細から共通化した `DetailsDialog` を使い、×・閉じる・背景クリック・Escで閉じて元の操作へフォーカスを戻す。再取得中は取得済み行を表示したまま更新し、初回取得失敗・前回一覧を保持する再取得失敗・未登録・検索不一致を区別する。日時は既存のサーバー時刻形式を維持し、不明は「記録なし」と表示する。

無効化・再有効化・ロール変更では `sessionVersion` を進め、更新トークンも削除する。再有効化しても無効化前のJWTは復活しない。パスワードのハッシュやソルトはJWT・操作履歴・検証エラーログへ含めない。JWTの資格情報タグにはセッション世代も反映するため、導入前のJWTは再ログインが必要になる。

### 管理者保護と操作履歴

- 自分自身の無効化・削除を拒否する。自分のパスワードは本人用APIで変更する。
- 最後の有効なADMINの無効化・削除・USERへの変更を拒否する。
- 管理操作はユーザー行をID順でロックし、最新状態と操作者の権限を再検証する。同時に管理者を降格しても最低1人を残す。
- `user_management_events` は日時、操作者、対象ユーザーID、操作種別、秘密値を含まない説明を保持する。作成、表示名変更、権限変更、無効化、有効化、パスワード再設定、本人変更、削除を記録する。
- 履歴は削除済みユーザーも参照でき、対象ユーザーIDの完全一致とページングで取得する。履歴をユーザーへの外部キーで連鎖削除しない。

### API

| API | 内容 |
| --- | --- |
| `POST /api/auth/change-password` | 本人のパスワード変更（認証済みUSER/ADMIN、変更専用JWTも可） |
| `GET /api/users` | `query`、`role`、`enabled`、`passwordChangeRequired` と既存ページング条件 |
| `PUT /api/users/{id}` | 既存項目に `enabled` を追加。パスワード指定は他ユーザーの再設定 |
| `GET /api/users/events` | `userId`（任意・完全一致）、`page`、`size`。ADMINのみ |

### DBと既存環境

ユーザーの指示により、既存ユーザーを検索して自動的に移行・補完する起動時処理や専用移行機能は追加しない。既存の `User` 監査列を再利用し、以下の列と操作履歴テーブルをスキーマに追加する。

| 追加列 | 型・初期値 |
| --- | --- |
| `enabled` | BOOLEAN NOT NULL。既存環境のスキーマ準備ではTRUE |
| `password_change_required` | BOOLEAN NOT NULL。既存環境のスキーマ準備ではFALSE、新規作成サービスはTRUE |
| `session_version` | BIGINT NOT NULL、0 |
| `last_login_at` | TIMESTAMP、NULL可 |
| `password_changed_at` | TIMESTAMP、NULL可 |

`ddl-auto=validate` の環境では、稼働前にこれらの列と `user_management_events`（エンティティ定義に対応するID・日時・操作者・対象・種別・詳細）を手動で用意する。既存のスキーマ更新設定を変更しない。DB操作前にバックアップを取得し、既存ユーザーのパスワード・権限・監査情報を保持する。未知のログイン日時・本人変更日時は捏造せずNULLとする。現在のAdminに必要な必須値の補完は運用作業として行い、自動移行のコードにしない。

## ライブラリ作成と設定

プロジェクトを親として基本情報と各設定JSONを管理する。変換プロファイル、ストレージ接続、テンプレート、ビューア、AI接続は別の登録情報を参照する。設定と実ファイルの保存先は同一とは限らない。

一般ユーザーの作成時は内部Web公開、既定の公開セキュリティ・ビューア、KEEP_PREVIOUSを適用する。開始・終了コマンドを除去し、管理用のパラメータ調整等を許可しない。選択肢は `GET /api/library-options` から取得し、管理接続の認証情報を返さない。変更可否はサーバーでも検証する。

## ゴミ箱と削除

`deletedAt`・`deletedBy` でゴミ箱状態を保持し、通常の検索・一覧・公開配信・新規実行対象から除外する。削除済みでも名前は予約される。待機・実行中ジョブや未完了の後続要求があるライブラリは削除できない。ライブラリ行のロックをジョブ予約とも共有する。

完全削除は設定レコードとパラメータ上書きを除去する。出力ファイル、ライブラリDB、ジョブ履歴を削除して容量を解放する操作ではない。操作履歴 `LibraryLifecycleEvent` は完全削除後も残る。

| API | 用途 |
| --- | --- |
| `GET /api/projects`、`GET /api/projects/{projectId}` | 閲覧可能プロジェクト |
| `GET/PUT /api/projects/{projectId}/members` | 管理者による参加者管理 |
| `GET/POST /api/projects/{projectId}/libraries` | ライブラリ一覧・作成 |
| `GET/PUT/DELETE /api/projects/{projectId}/libraries/{libraryId}` | 詳細・更新・ゴミ箱移動 |
| `GET/DELETE /api/projects/{projectId}/libraries/trash` | ゴミ箱一覧・空にする |
| `POST /api/projects/{projectId}/libraries/{libraryId}/restore` | 復元 |
| `DELETE /api/projects/{projectId}/libraries/{libraryId}/permanent` | 完全削除 |

## インポート・エクスポート

プロジェクト単位とライブラリ設定単位の入出力は管理者操作。`GET /api/projects/{projectId}/export`、`POST /api/projects/import`、`POST /api/libraries/export`、`POST /api/libraries/import` を使用する。成果物、DB、認証情報のバックアップとは区別する。入力DTO・入出力Serviceの検証を通して設定を復元する。

文書ごとの公開・ダウンロード・印刷・コピー等は `DocumentMetaController` と実効設定解決に従う。公開状態・期間・権限の判定は [公開閲覧](search-and-viewing.md) を参照。

## 根拠

- [SecurityConfig](../../backend/src/main/java/jp/astack/backend/config/SecurityConfig.java)、[ProjectAccessService](../../backend/src/main/java/jp/astack/backend/security/ProjectAccessService.java)
- [LibraryServiceImpl](../../backend/src/main/java/jp/astack/backend/service/LibraryServiceImpl.java)、[Library](../../backend/src/main/java/jp/astack/backend/model/entity/Library.java)
- [ProjectImportExportController](../../backend/src/main/java/jp/astack/backend/controller/ProjectImportExportController.java)、[LibraryManagementController](../../backend/src/main/java/jp/astack/backend/controller/LibraryManagementController.java)
- [ゴミ箱回帰テスト](../../backend/src/test/java/jp/astack/backend/service/LibraryServiceTrashTest.java)

### 未対応ファイルの公開設定

ライブラリの「文書設定 → 公開設定」に、コンバーター未対応形式の公開／非公開と、非公開時に例外として公開する拡張子を設ける。既定は非公開、例外なし。管理者だけが変更でき、インポート・エクスポートでも保持する。詳細は [ジョブと公開](jobs-and-publication.md#コンバーター未対応ファイルの公開) を参照。

## ログイン試行制限（#6728）

`POST /api/auth/login` はDBロック・BCrypt実行前に、ID（大小文字を同一集計）・物理接続元・インスタンス全体で入場制御する。存在しないIDも同じ規則。`Forwarded` / `X-Forwarded-For` は採用せず、Tomcatが保存したソケット接続元を使用する。プロキシ配下ではプロキシ単位の集計となるため、プロキシ側でも信頼境界を確定したクライアント別制限を併用する。

既定60秒窓でID 20回、接続元100回、全体500回まで。同時認証はID 1件、接続元4件、全体16件。IDで5失敗、接続元で20失敗以降は1/2/4/8/16/30秒の段階的待機を設ける（窓の終了まで）。成功時はIDの失敗待機を解除し、要求総数は維持する。拒否時は共通のHTTP 429、`Retry-After` 秒数、`LOGIN_RATE_LIMITED`、`Cache-Control: no-store` を返す。拒否要求は待機や窓を延長しない。恒久的なユーザー無効化はしない。

`astack.login-limit.account-budget` / `source-budget` / `global-budget` / `window-seconds` / `capacity` を正の値で設定できる。capacityはID・接続元それぞれ既定10000件。満杯時は新規の認証を拒否し、進行中・未期限の項目を追い出して制限を回避させない。

試行制限で拒否された要求は新しいアカウント・送信元の管理エントリーを作成しない。送信元、アカウント、全体同時数の制限中にランダムID等を送信しても、管理領域を消費して別ユーザーのログインを妨げない。

## 保存後のビルド案内

初期登録・設定更新後はサーバーが未反映状態を判定し、必要なビルドと理由を確認する。見送った場合も一覧から実行できる。分類、API、移行上の扱いは [ライブラリ設定とビルドの対応表](library-build-requirements.md) を参照。
