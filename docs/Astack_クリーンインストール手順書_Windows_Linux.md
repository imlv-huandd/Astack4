# Astack Version 4 クリーンインストール導入手順書

## 文書管理情報

- 文書番号: ASTACK4-OPS-INS-001
- 文書名: Astack Version 4 クリーンインストール導入手順書
- 版数: 1.0
- 作成日: 2026-05-11
- 対象製品: Astack Version 4
- 対象OS: Windows / Linux
- 配布区分: 社内配布用
- 想定読者: 導入担当者、インフラ運用担当者、保守担当者

## 1. 目的

本書は、Astack を Windows または Linux のクリーンインストール環境へ新規導入する際の標準作業手順を定義するものです。

本書の対象範囲は以下です。

- Astack アプリケーション本体の配置と起動
- 組み込み H2 データベース利用時の初期導入
- OpenSearch の接続前提と初期連携確認
- 文書変換エンジン連携の初期補正

本書は 2026-05-11 時点のリポジトリ実装に基づいています。現行版では、Windows 側は自動インストーラを利用でき、Linux 側は手動導入を前提とします。

## ライセンスと30日体験版

初回導入では30日間の自動体験版（Lite相当）が開始します。WindowsインストーラーはID・開始日時・期限を `%ProgramData%\Astack\license` に保存し、通常アンインストール後も保持します。Linux手動導入では `/var/lib/astack/license` など削除されない領域を用意し、サービスの `ASTACK_LICENSE_DIRECTORY` に設定してください。実行ユーザーに読み書き権限が必要です。既存導入は従来のID・ライセンス一式をバックアップして移行します。

フルトライアルや正式購入版は、ライセンス管理画面のインストールIDを提出して発行を受け、同画面で登録します。OSDCライセンスを同梱する場合はOSと製品バージョンが一致するファイルを使用します。詳細は [自動体験版とOSDC同梱](specs/trial-and-osdc-bundling.md) を参照してください。

## 2. 本書の利用方針

- 本書はクリーンインストール環境への新規導入時に使用します。
- 既存環境の更新、移行、切戻し手順は本書の対象外です。
- 本書に記載の初期値を使用した場合は、導入後に必ず認証情報と接続設定を見直してください。

## 3. 先に把握しておく制約

現行実装には、クリーンインストール時に重要な制約があります。導入前に必ず確認してください。

### 3.1 Windows 側は自動インストーラあり、Linux 側は手動導入

- Windows は installer/Windows 配下の setup.bat / install.ps1 で導入できます。
- Linux は同等の install.sh がないため、手動で OpenSearch と Astack サービスを構成します。

### 3.2 初回管理者ユーザーは自動生成される

- 初回起動時、ユーザーID `admin` が存在しない場合は管理者ユーザーが自動生成されます。
- ユーザー ID は admin です。
- `INIT_ADMIN_PASSWORD` が未設定の場合はランダムな初期パスワードを生成し、`${astack.data-path}/secrets/initial-admin-password` へ保存します。固定値の admin ではありません。
- Windows の既定保存先は `C:\Astack\app\data\secrets\initial-admin-password` です。サーバー管理者が確認し、秘密値をログや問い合わせへ転記しないでください。
- Windowsインストーラーの完了画面には、ユーザーID `admin` と生成された初期パスワードを表示します。非対話実行または標準出力のリダイレクト時は値を表示せず、保存先のみ案内します。
- 初回ログイン後、必ず管理者パスワードを変更してください。

### 3.3 デフォルトのコンバーターエンジン設定は配布環境向けではない

- 初回起動時に登録されるデフォルトコンバーターエンジンは、開発ワークスペース上の絶対パスを向いています。
- クリーンインストール環境ではそのまま使えません。
- 導入後に、コンバーターエンジンの実行パスを実サーバー上のパスへ更新する必要があります。

### 3.4 Windows インストーラは OpenSearch 連携設定を自動補正する

- Windows の install.ps1 は導入ごとに暗号学的乱数からOpenSearch管理者パスワードを生成します。値は `<インストール先>\secrets\opensearch-admin-password` とサービスXMLに保存し、SYSTEMとAdministratorsだけがアクセスできるACLを設定します。ログや問い合わせへ値を転記しないでください。
- あわせて、AstackServer サービスへ SEARCH_OPENSEARCH_HOST / PORT / SCHEME / USERNAME / PASSWORD を自動設定します。
- RESTはHTTPのまま、HTTP/transport双方を `127.0.0.1` に固定します。別ホストからの接続には使用できません。リモート利用には別途TLSとネットワーク制限の設計が必要です。
- 既存OpenSearchがある場合は展開前に停止します。更新時は[資格情報の移行・検証](windows-opensearch-security.md)を参照してください。
- そのため、インストール直後の確認は `http://127.0.0.1:9200` に対して実施してください。

### 3.5 現行の本番プロファイルはクリーンインストール向けではない

- application-prod.yml は H2 TCP 接続と ddl-auto=validate を前提にしています。
- スキーマが未作成の新規環境では、そのままでは起動できません。
- 現行版のクリーンインストールでは、まず既定プロファイルで導入し、必要に応じて後続で本番向け構成へ移行してください。

## 4. 共通の事前準備

### 4.1 必要なソフトウェアと配布物

共通で以下を用意します。

- Astack.jar
- OpenSearch 2 系の配布物
- Java 25 実行環境
- 文書変換を利用する場合は Antenna House Office Server Document Converter

ソースから Astack.jar を作成する場合は、発行システムから取得した公開鍵JSONと、別途承認した公開鍵のSPKI指紋を指定します。ライセンス鍵の承認・指紋確認については[ライセンス仕様](specs/licensing.md)を参照してください。

```bash
cd /workspaces/astack4/build
./build-jar.sh --license-keys /path/to/trusted-keys.json \
  --license-key-id KEY_ID --license-key-sha256 APPROVED_SHA256
```

生成物は build/Astack.jar です。

### 4.2 推奨ポート

標準では以下を使用します。

- Astack: 8080
- OpenSearch: 9200

ファイアウォールやセキュリティグループで必要に応じて許可してください。

### 4.3 推奨ディレクトリ方針

Astack は相対パスで data と log を作成するため、サービスの作業ディレクトリを固定してください。

推奨例:

- Windows: C:\Astack
- Linux: /opt/astack/app

## 5. Windows クリーンインストール手順

### 5.1 前提条件

- Windows Server 2022 / 2025 または Windows 10 / 11 x64
- 管理者権限でログオンできること
- PowerShell 5.1 以上
- インストール媒体内に installer/Windows/payloads があり、以下が配置されていること
  - OpenJDK*.zip
  - opensearch-*.zip
  - Astack.jar
  - WinSW*.exe
  - vc_redist.x64.exe（OSDC用のMicrosoft Visual C++ 再頒布可能パッケージ x64）

配布物の作成時は、Microsoftから取得したOSDC対応版の `vc_redist.x64.exe` を、他の必須ペイロードと同じ `--required-payloads-dir` の直下へ配置してください。再頒布のライセンス条件とMicrosoftのデジタル署名を確認したファイルを使用します。インストール時のダウンロードは行いません。

### 5.2 インストール媒体の配置

インストール媒体を任意の作業フォルダへ展開します。例:

```powershell
C:\temp\astack-installer
```

以下の構成になっていることを確認します。

```text
setup.bat
install.ps1
payloads\OpenJDK....zip
payloads\opensearch-....zip
payloads\Astack.jar
payloads\WinSW....exe
payloads\vc_redist.x64.exe
```

### 5.3 自動インストール実行

1. エクスプローラーまたは管理者のコマンドプロンプトから setup.bat を実行します。
2. フォルダ選択ダイアログでインストール先を選択します。
3. install.ps1 により以下が自動実行されます。
   - ディレクトリ作成
   - Visual C++ 再頒布可能パッケージ x64 のサイレントインストール（`/install /quiet /norestart`）
   - JDK 展開
   - OpenSearch 展開
  - payloads 配下の追加 zip 展開
   - Astack.jar 配置
   - AstackSearch サービス登録
   - AstackServer サービス登録
4. 完了画面に表示されるユーザーID `admin` と初期パスワードを確認し、初回ログインしてパスワードを変更します。その後、完了画面を閉じます。

既定のインストール先は C:\Astack です。

Visual C++ ランタイムはサービス起動前に導入します。終了コード `0` は成功、`3010` は成功・再起動要求として扱い、自動再起動は行いません。`3010` の場合は完了画面の案内に従い、Windowsを再起動してからOSDCを使用してください。既存の別バージョンを示す `1638`（HRESULT形式では `-2147023258`）の場合は既存ランタイムを使用し、それ以外の終了コードや起動失敗ではインストールを中止します。実行結果は `<インストール先>\services\logs\install.log`、ランタイムの詳細ログは同ディレクトリの `vc_redist.x64.log` に記録します。ランタイムは共有コンポーネントのためAstackのアンインストール時には削除しません。

完了時は初期パスワードの生成を最大120秒待機します。生成途中の空ファイルや不完全な値は表示しません。未生成・読み取り失敗の場合は警告と保存先を案内します。サービスのログで起動状態を確認し、生成後に管理者が保存先を確認してください。`INIT_ADMIN_PASSWORD` を明示した場合はファイルが生成されないため、指定した値を使用します。

インストーラーはインストールログにユーザーID `admin` と初期パスワードファイルの保存先を記録し、パスワード自体は記録しません。ただし、画面録画・PowerShellのトランスクリプト等には表示内容が残る可能性があるため、記録・共有を避けてください。表示対象はクリーンインストール時の初期値であり、変更後のパスワードではありません。初回ログインでは変更が必須です。

### 5.4 インストール直後の確認

Windows 自動インストーラは、Astack と OpenSearch の連携に必要な認証値と通信方式を自動設定します。以下を確認してください。

アプリの秘密情報とサポート用一時ファイルは、サービス開始前に `<インストール先>\app\data\secrets` および `<インストール先>\app\data\support-temp` をSYSTEM・Administrators限定にして保護します。新規ファイルは親ACLを継承し、Astackは起動時にWindowsアカウント名の解決やACLの検査・変更を行いません。手動導入・保存先変更では、秘密情報の生成前にOS管理者が両フォルダを準備・保護してください。既存環境の更新にこのクリーンインストーラーを使用しないでください。実行アカウント変更時の手順は [管理者マニュアル](Astack_管理者マニュアル_Windows_Linux.md#311-jwt署名鍵の管理) を参照してください。

1. `C:\Astack\services\AstackServer.xml` に `SEARCH_OPENSEARCH_HOST=127.0.0.1`、`SEARCH_OPENSEARCH_PORT=9200`、`SEARCH_OPENSEARCH_SCHEME=http`、`SEARCH_OPENSEARCH_USERNAME=admin` が設定され、パスワードが導入ごとの秘密ファイルと一致することを管理者が確認します。値をコピーしてログ・履歴・問い合わせへ残さないでください。

2. 管理者PowerShellで、秘密をコマンド引数へ展開せず認証付き疎通を確認します。

```powershell
$searchPassword = [IO.File]::ReadAllText('C:\Astack\secrets\opensearch-admin-password')
$searchAuth = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('admin:' + $searchPassword))
try {
    Invoke-RestMethod -Uri 'http://127.0.0.1:9200/_cluster/health' -Headers @{ Authorization = 'Basic ' + $searchAuth }
} finally {
    Remove-Variable searchPassword, searchAuth -ErrorAction SilentlyContinue
}
```

3. status が green または yellow で返ることを確認します。

注意:

- OpenSearch の管理者パスワードを別値へ変更した場合は、`C:\Astack\secrets\opensearch-admin-password` と `C:\Astack\services\AstackServer.xml` の SEARCH_OPENSEARCH_PASSWORD も同じ値へ変更し、ACLを確認してAstackServerサービスを再起動してください。手動起動用run-Astack.batもこの秘密ファイルを読み込みます。詳細は[移行・復旧手順](windows-opensearch-security.md)を参照してください。
- OpenSearch のログに NotSslRecordException が出る場合は、HTTPS ではなく HTTP で確認しているか、または過去のインストール物が混在していないかを確認してください。
- 初回ログインには、生成された初期パスワードまたは明示した `INIT_ADMIN_PASSWORD` を使用し、ログイン後に必ず変更してください。

### 5.5 動作確認

PowerShell でサービス状態を確認します。

```powershell
Get-Service AstackSearch, AstackServer
```

両方が Running であることを確認します。

初回ビルド後に、Astack のログへ「インデックス登録でエラー」が出ていないことも確認してください。

ブラウザで以下へアクセスします。

```text
http://localhost:8080
```

初回ログイン情報:

- ユーザー ID: admin
- パスワード: `C:\Astack\app\data\secrets\initial-admin-password` の値（`INIT_ADMIN_PASSWORD` を明示した場合はその値）

### 5.6 ログと主要ファイル

- インストールログ: C:\Astack\services\logs\install.log
- Astack ログ: C:\Astack\services\logs
- Astack 実体: C:\Astack\app\Astack.jar
- OpenSearch: C:\Astack\OpenSearch
- アンインストーラ: C:\Astack\uninstall.bat

### 5.7 導入後に必ず行う作業

1. admin ユーザーのパスワードを変更する
2. コンバーターエンジンの実行パスを実サーバーの実行ファイルへ変更する
3. ストレージ接続、テンプレート、ライブラリを登録する
4. 必要に応じて OpenSearch 管理者パスワードを変更し、AstackServer.xml も追従させる

## 6. Linux クリーンインストール手順

### 6.1 前提条件

- systemd が利用できる Linux
- sudo 権限
- Java 25 実行環境
- OpenSearch 2 系配布物
- 文書変換を利用する場合は Antenna House Office Server Document Converter

Office Server Document Converter は、同梱 ReadMe.txt の通り RPM インストール前提です。Linux_X86_64J を単純コピーして使う場合は、実行権限とライブラリ解決に注意してください。

### 6.2 ディレクトリ作成

以下では /opt/astack 配下へ配置します。

```bash
sudo mkdir -p /opt/astack/app
sudo mkdir -p /opt/astack/opensearch
sudo mkdir -p /opt/astack/converter
sudo useradd -r -s /usr/sbin/nologin astack || true
sudo chown -R astack:astack /opt/astack
```

### 6.3 Astack.jar 配置

生成済みの Astack.jar を配置します。

```bash
sudo cp /workspaces/astack4/build/Astack.jar /opt/astack/app/Astack.jar
sudo chown astack:astack /opt/astack/app/Astack.jar
```

### 6.4 OpenSearch 配置

OpenSearch の導入方法は運用標準に合わせて構いませんが、Astack と同一ホストへ入れる場合は localhost:9200 で応答するように構成してください。

例として tar 配布物を /opt/astack/opensearch へ展開する場合:

```bash
sudo tar -xf opensearch-2.x.x-linux-x64.tar.gz -C /opt/astack/opensearch --strip-components=1
sudo chown -R astack:astack /opt/astack/opensearch
```

OpenSearch の初期管理者パスワードは、任意の強固な値を採用してください。以降の Astack サービス設定でも同じ値を使います。

### 6.5 変換エンジン配置

推奨はベンダー提供 RPM による導入です。RPM で導入できる場合はそれを優先してください。

同梱の Linux_X86_64J を利用する場合の例:

```bash
sudo cp -a /workspaces/astack4/Linux_X86_64J/. /opt/astack/converter/
sudo chown -R astack:astack /opt/astack/converter
sudo chmod +x /opt/astack/converter/run.sh
sudo chmod +x /opt/astack/converter/bin/SBCCmd
```

日本語を含む原本ファイルを PDF 変換する場合は、OSDC 実行環境へ日本語フォントを必ず配置してください。未配置のままでは出力 PDF が文字化けします。

Debian/Ubuntu 系で Noto 系フォントを追加する例:

```bash
sudo apt-get update
sudo apt-get install -y fonts-noto-cjk
```

現行の Linux 同梱テンプレートは Noto Serif CJK JP / Noto Sans CJK JP を既定の日本語フォントとして参照します。OS 同梱フォントを使わず個別配布する場合は、ライセンス条件を確認した上で Noto Serif CJK JP / Noto Sans CJK JP を /opt/astack/converter/fonts へ配置してください。
IPAex 系フォントしかない環境では、/opt/astack/converter/etc/sbc-opt.xml の script-font / font-alias を IPAex 系へ変更してください。

補足:

- run.sh は起動時に LD_LIBRARY_PATH と SBC110 系環境変数を設定します。
- run.sh は lib 配下の SONAME 用シンボリックリンクを補完します。
- run.sh は日本語フォントが見つからない場合に標準エラーへ警告を出します。
- 単純コピー環境ではライブラリシンボリックリンク不足や実行権限不足で失敗しやすいため、初回確認を必ず行ってください。

### 6.6 systemd ユニット作成

#### 6.6.1 Astack サービス

以下のユニットファイルを作成します。

ファイル: /etc/systemd/system/astack.service

```ini
[Unit]
Description=Astack Server
After=network.target opensearch.service
Wants=opensearch.service

[Service]
Type=simple
User=astack
Group=astack
WorkingDirectory=/opt/astack/app
Environment=INIT_ADMIN_PASSWORD=ChangeMe-Immediately
Environment=SEARCH_OPENSEARCH_USERNAME=admin
Environment=SEARCH_OPENSEARCH_PASSWORD=ChangeThisOpenSearchPassword
Environment=JWT_SECRET=64文字以上の十分に長いランダム文字列
ExecStart=/usr/bin/java -jar /opt/astack/app/Astack.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

重要:

- 現行版のクリーンインストールでは、--spring.profiles.active=prod を付けずに起動してください。
- 初回起動で INIT_ADMIN_PASSWORD を設定しておくと、admin ユーザーの初期パスワードを固定できます。

#### 6.6.2 OpenSearch サービス

OpenSearch を OS 標準のパッケージや別の systemd ユニットで導入済みであれば、その既存サービスを利用してください。

少なくとも以下を満たしてください。

- localhost:9200 で待受していること
- Astack から admin ユーザーで接続できること

### 6.7 サービス起動

```bash
sudo systemctl daemon-reload
sudo systemctl enable astack
sudo systemctl start astack
sudo systemctl status astack --no-pager
```

OpenSearch も未起動であれば先に起動してください。

### 6.8 動作確認

HTTP 応答確認:

```bash
curl -I http://localhost:8080
```

ブラウザ確認:

```text
http://localhost:8080
```

初回ログイン情報:

- ユーザー ID: admin
- パスワード: astack.service に設定した INIT_ADMIN_PASSWORD の値

### 6.9 導入後に必ず行う作業

1. admin ユーザーでログインし、パスワードを変更する
2. コンバーターエンジンの実行パスを実サーバーのパスへ変更する
3. OpenSearch 接続確認を行う
4. ストレージ接続、テンプレート、ライブラリを登録する

コンバーターエンジンの設定例:

```text
/opt/astack/converter/run.sh -d {input} -o {output} -i /opt/astack/converter/etc/sbc-opt.xml
```

## 7. 共通の初回設定

導入後、OS を問わず以下を実施してください。

### 7.1 管理者パスワード変更

初回の admin ユーザーは自動生成です。必ず運用用の強固なパスワードへ変更してください。

### 7.2 コンバーターエンジン設定見直し

デフォルト登録されるエンジン設定は開発環境のパスを向くため、そのままでは利用できません。実際に導入した変換エンジンの実行パスへ変更してください。

### 7.3 OpenSearch 接続確認

検索機能を使う場合は、Astack と OpenSearch の認証値が一致していることを確認してください。

確認観点:

- OpenSearch URL が localhost:9200 または実運用 URL であること
- OpenSearch の認証情報が Astack サービス設定と一致していること
- Astack 起動ログに OpenSearch 接続エラーが出ていないこと

### 7.4 監視対象の確認

最低限、以下を監視対象にしてください。

- Astack プロセス稼働
- OpenSearch プロセス稼働
- data ディレクトリ容量
- log ディレクトリ容量

## 8. 動作確認チェックリスト

以下を全て満たせば、導入完了と判断できます。

- ブラウザで http://localhost:8080 が開く
- admin でログインできる
- ダッシュボードが表示される
- OpenSearch に接続エラーが出ていない
- コンバーターエンジン設定が実サーバーパスへ更新済み
- ライブラリ設定画面が開ける

## 9. トラブルシューティング

### 9.1 admin でログインできない

原因候補:

- INIT_ADMIN_PASSWORD の認識違い
- 既存 H2 データファイルが残っている

対処:

- data 配下に既存 DB がないか確認する
- 新規環境であれば、Astack 停止後に data を退避して再起動する

### 9.2 検索が失敗する

原因候補:

- OpenSearch サービス未起動
- OpenSearch パスワード不一致
- Astack サービスへ SEARCH_OPENSEARCH_PASSWORD が渡っていない

対処:

- OpenSearch の応答を curl で確認する
- Astack のサービス定義へ OpenSearch 認証情報を設定する

### 9.3 変換に失敗する

原因候補:

- デフォルトコンバーターエンジンが開発環境パスのまま
- Linux で run.sh に実行権限がない
- Linux で変換ライブラリのシンボリックリンクが不足している

対処:

- コンバーターエンジンの実行パスを修正する
- chmod +x を再実行する
- OSDC は可能なら RPM で再導入する

### 9.4 Linux で本番プロファイル起動に失敗する

原因候補:

- H2 TCP サーバー未構成
- スキーマ未作成なのに ddl-auto=validate で起動している

対処:

- 現行版は既定プロファイルで導入する
- 本番プロファイル化する場合は、別途 H2 サーバーとスキーマ初期化手順を整備してから切り替える

## 10. 改訂履歴

- 2026-05-11 初版作成
- 2026-06-01 Windows インストーラーの OpenSearch 自動補正と確認手順を反映