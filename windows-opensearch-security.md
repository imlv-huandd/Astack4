# Windows OpenSearch資格情報・待ち受けの保護

対象: #6721、2026-10-26。正本は `installer/Windows/install.ps1`。

## 新規導入

48バイトの暗号学的乱数と複雑性条件の接頭辞から導入ごとのパスワードを生成する。秘密値を出力せず、`secrets/opensearch-admin-password` と `services/AstackServer.xml` に保存する。親ディレクトリは継承ACLを除去し、SYSTEMとAdministratorsに制限する。ACL設定失敗時は中止する。サービスは既存構成のLocalSystemで動作する。

AstackServerの接続先も `SEARCH_OPENSEARCH_HOST=127.0.0.1`、port=9200、scheme=httpを明示する。手動起動用 `run-Astack.bat` は毎回秘密ファイルを読み込み、同じ接続先・資格情報をJava子プロセスへ渡す。秘密値をbatへ埋め込まない。遅延展開を無効化して `!` を保持し、変数はsetlocalの範囲に限定する。秘密ファイルが未存在・空・読み取り不可ならJavaを起動しない。

REST TLSを無効にする構成では `network.host`・`http.host`・`transport.host` を `127.0.0.1` に固定し、単一ノードにする。別途指定されたbind_host/publish_hostを残さず、HTTP 9200/transport 9300を明示する。ローカル管理者からの読み取りや同一ホスト内の盗聴を防ぐものではない。デモ用transport証明書を本番向けTLS構成と見なさない。

## 既存導入の移行・復旧

このクリーンインストーラーで既存OpenSearchを更新しない。既存ディレクトリを検出したら、展開・削除前に中止する。秘密ファイルだけが残った再試行も上書きせず中止する。

1. AstackServerを停止し、OpenSearchスナップショット、設定、サービスXML、ACLを管理者専用の場所へバックアップする。OpenSearchのデータフォルダを削除しない。
2. OpenSearchの版に合うSecurity APIのパスワード変更手順を使用して、管理者パスワードを新しい強い値へ変更する。変更をシェル引数、履歴、ログに残さない。単に環境変数を変えて再起動しても既存Securityインデックスの資格情報は更新されない。
3. 新しい秘密値を上記ファイルとAstackServer.xmlへ設定し、保存先ACLをSYSTEM/Administrators限定にする。既存ファイルに追加の明示ACLがないことも確認する。
4. OpenSearch設定をloopbackへ変更して再起動する。旧値で401、新値で正常応答を確認してからAstackServerを起動する。
5. 失敗時はサービスを停止し、設定と資格情報を同じ世代へ戻す。Securityインデックス内のパスワードとXMLを不一致にしない。復旧のために全インターフェイス待ち受けや固定値へ戻さない。

## 専用アカウントと残課題

Astackは全文/ベクトル/履歴のインデックス作成、マッピング更新、検索・書き込み・削除、クラスタ診断を使用する。今回の変更では既存のadmin接続を維持する。専用アカウントはインデックスprefix、クラスタAPI、Spring AIによる初期化を含め実機で権限を確認してから導入する必要がある。管理者の常用回避は未実装であり、本チケットは完了扱いにしない。

## 実機を使わない構成検証

`pwsh -NoProfile -File installer/Windows/tests/Test-OpenSearchConfiguration.ps1` で、インストーラーの構文、合成値によるサービスXML・バッチの生成、設定値の一致と秘密値の非埋込み、YAML補正ブロックの動作を検証する。YAMLは一時ディレクトリだけを書き換える。実インストーラー・サービス・ネットワーク・ACL処理は実行しないため、Windows実機検証を代替しない。

## Windows実機で必要な検証

このLinux作業環境ではWindowsサービス・ACL・OpenSearchを実行していない。以下は未検証。

- 異なる新規導入2件で秘密値が異なり、過去の固定値では401になること。
- `Get-NetTCPConnection` で9200/9300のLocalAddressが127.0.0.1だけであること、別ホストから接続できないこと。
- `icacls` と一般ユーザーで秘密ファイル/サービスXMLを読めないこと。
- 正規の全文検索、AIベクトル、アクセス履歴、再起動、手動の移行/復旧が動作すること。サービス起動とrun-Astack.batの手動起動が同じ資格情報で接続できること、空・未存在・アクセス拒否の秘密ファイルでは手動起動を拒否すること。
- 初期設定失敗/ACL失敗/既存導入検出時に停止して既存データを破壊しないこと。
