# ストレージ接続と公開能力

確認基準は [索引](../README.md) を参照。

## 登録済み方式

| 方式 | 公開戦略 | 既定 |
| --- | --- | --- |
| LOCAL | DIRECT / STAGING | STAGING |
| CIFS、FTP、FTPS、SFTP | DIRECT / STAGING | DIRECT |
| S3 | DIRECT | DIRECT |

`INTERNAL_HTTP` は接続登録方式ではなく、内部Web公開先をLOCALへ解決するポリシー。方式と設定型、生成処理、能力は `StorageProviderRegistry` に登録する。列挙値だけを見て対応済みと判定しない。

ライブラリの入力先・公開先・既定公開先を `LibraryStorageResolver` で解決する。メインDBの接続情報はライブラリDBへの切替前に解決する。Providerを処理間で共有しない。

## 共通契約

- 接続はセッション確立。入力ルートを自動作成しない。READ_DIRECTORYは存在・一覧、WRITE_DIRECTORYは出力先としての準備を確認する。
- 一覧は直下の要素を返し、パスはルート相対。取得失敗を正常な空一覧へ置換しない。完全走査が成功してから削除判定を行う。
- 親参照・絶対パス・制御文字を拒否し、LOCALでは実パスとシンボリックリンクのルート境界を検査する。
- 原本取得は `StorageFileLease` を使用し、リモート一時ファイルを応答完了・切断のいずれでも解放する。
- 除外は全方式でglobの全体一致。`*` と `?` を使用し、`.` は文字どおり。CIFSの旧部分一致は使わない。

接続テストは接続と一覧取得を確認する。未保存テストでファイルを作成せず、書込権限は未確認と表示する。保存済み秘密値のnull・マスク値は同方式の既存値を維持し、空文字は削除要求として検証する。新規や方式変更時の未解決マスクは拒否する。

## 方式ごとの条件

FTP/FTPSはバイナリ転送、passive、encoding、timeoutを反映する。FTPSはTLSとデータ接続保護、証明書・ホスト名を検証する。サーバーが通知しない物理リンクの隔離はFTPサーバーのユーザールート設定による。

SFTPはknown_hostsによるSTRICTを既定とし、ACCEPT_ALLは明示設定のみ。パスワード認証・秘密鍵認証を持つ。ADD_NEWとPASSWORD_AND_KEYは保存時に拒否する。

S3はendpoint、region、資格情報、`pathStyleAccess` とprefix境界を共通利用する。仮想フォルダと一覧ページングを扱う。STAGINGへ自動変更しない。

## 公開と追加方式

公開戦略はProviderの `PublicationSupport` に委譲し、非対応戦略は公開先の変更前に拒否する。ディレクトリ移動ができても複数ファイル全体の原子性を一律には保証しない。差分公開はDIRECTマージと削除整理を行う。

方式追加は設定型・JSON subtype、Provider、`StorageProviderRegistration` のBean、フロントエンド `protocolFormRegistry` を追加する。共通Factory・ビルド・接続テストへ方式別switchを増設しない。契約テストに加え実通信で入力・公開を検証する。

代表APIは `/api/storage-connections` のCRUD、`GET /protocols`、`POST /test`（未保存）、`POST /{id}/test`（保存済み）。能力・秘密情報を含まない表示先もレスポンスで提供する。

## 根拠

- [登録方式と能力](../../backend/src/main/java/jp/astack/backend/build/storage/StorageProviderConfiguration.java)、[Registry](../../backend/src/main/java/jp/astack/backend/build/storage/StorageProviderRegistry.java)
- [StorageProvider](../../backend/src/main/java/jp/astack/backend/build/storage/StorageProvider.java)、[LibraryStorageResolver](../../backend/src/main/java/jp/astack/backend/build/storage/LibraryStorageResolver.java)
- [StorageConnectionController](../../backend/src/main/java/jp/astack/backend/controller/StorageConnectionController.java)、[StorageConfigurationValidator](../../backend/src/main/java/jp/astack/backend/service/StorageConfigurationValidator.java)
- [拡張時の詳細手順・旧記録](../history/designs/storage-extension-guide.md)

## 接続設定のDB暗号化（#6730）

`storage_connection.connection_json` 全体を、AI接続と同じ `SecretsCryptoService` のAES-256-GCM（`enc:v1`）で暗号化する。追加プロトコルの秘密フィールドも対象となる。JPAの読出し時だけ復号し、既存のAPIマスク・マスク値再保存・接続/クロール/公開経路を維持する。DBの生JSONを直接参照する外部ツールは暗号文を受け取る。

新規導入を前提とし、旧平文レコードの一括暗号化移行は提供しない。新規登録・更新時にJPAコンバーターで暗号化する。起動時にはAI接続とストレージ接続の保存済み暗号文を読み取り専用で検証し、DB更新や暗号化移行は行わない。鍵欠損・不一致時は起動を中断し、新しい鍵で上書きしない。
