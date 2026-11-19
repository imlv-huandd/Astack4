# ライセンス管理と利用制限

自動30日体験版・OSDC同梱の追加仕様は [自動体験版とOSDCライセンス](trial-and-osdc-bundling.md)、登録時ホスト名を含む新規署名形式は [v5契約](../contracts/license-v5.md) を参照。v5は製品バージョン範囲を [v4契約](../contracts/license-v4.md)、OSDC同梱形式を [v3契約](../contracts/license-v3.md) から継承する。

関連: 登録時ホスト名 [#6811](https://developer.astack.jp/redmine/issues/6811)、端末紐付け [#6809](https://developer.astack.jp/redmine/issues/6809)、[#6714](https://developer.astack.jp/redmine/issues/6714)、基盤 [#6713](https://developer.astack.jp/redmine/issues/6713)、設計 [PR #49](https://github.com/nt-bsys-dev/Astack4/pull/49)。本書は製品統合ブランチの実装仕様。mainへの取り込み・本番適用済みを意味しない。

## 確定した動作と提供範囲

- インストールIDに紐付いた署名済みファイルを製品内で検証する。通常起動・利用時に発行サーバーへ接続しない。移設時は再発行する。
- 期限切れ後も既存公開文書の閲覧・原本ダウンロード・短縮URL解決を継続する。従来の認証、ACL、公開期間、ゴミ箱等の条件は維持する。
- 未登録・開始前・無効・期限切れでは、新規ビルド、変換開始、更新公開、AI処理、検索/索引更新を停止する。参照・キャンセル・認証・パスワード変更・ライセンス復旧経路を維持する。
- 待機ジョブと後続要求は権利がない間に消費しない。ワーカーの実行直前に失効したRUNNINGジョブはWAITINGへ戻す。すでに進行した処理が後段の検査で拒否された場合は既存の失敗/取消処理で終了し、権利回復後に再実行する。
- ログイン状態・管理者権限・業務上の所属判定をライセンスで代替しない。プロジェクト・ライブラリ・文書の増加にサーバー側で上限を適用する。ユーザー数はライセンス枠に含めない。

販売オプションは「全文検索」「AI検索」「アクセス履歴」「Astack Converter Plugin」「ドキュメントポータルキット」。初期エディションはStandard/Lite/体験版とし、発行側の管理画面でエディションごとの既定権利・利用枠を設定する。製品はエディション名から権利を推測せず、署名された `features` と `limits` を使用する。新規の製品バージョン指定発行は、登録時のホスト名を署名する [v5契約](../contracts/license-v5.md) に従う。既存v4の形式は維持する。既存の [v2契約](../contracts/license-v2.md)・[v3契約](../contracts/license-v3.md) の検証も維持し、有効なライセンスに基本ビルド権利を含める。

`CONVERTER_PLUGIN` は、初期登録の `astack-converter-osdc` エンジンによるACP実変換が今回必要かを、予約時と実行開始時に検証する（[#6733](https://developer.astack.jp/redmine/issues/6733)）。権限不足なら手動・API予約は理由付き入力エラーで拒否し、スケジュール実行は理由・終了日時を持つFAILEDジョブを記録する。開始時の検査は公開先の作成・スナップショット更新・原本登録・変換より前に行い、予約後に原本や権限が変わった場合も再検査する。入力ストレージの読取り失敗時は確認不能のエラーとし、実行可能とは扱わない。

TEMPLATE、他エンジン、原本除外・公開対象外・未対応形式、設定により外部エンジンを使わない同形式コピーにはACP権限を要求しない。DIFFで原本と変換設定が一致し保存済み成果物を再利用する場合も不要。FULLで実変換が必要なら再利用によって検査を省略しない。表示名ではなくpluginIdで判定する。

実行中は投入前・外部エンジン起動直前とビルドの進捗確認時（2秒間隔）に再検査し、ACPの投入待ち・実行中に権限を失ったらビルドをFAILEDにして変換を中止し、公開・AI後続予約を行わない。権限不足の変換ジョブはリトライせずFAILEDにし、理由を画面のジョブ詳細とビルドログへ残す。無期限のACP権利待ちとタイムアウト除外は廃止する。旧版のACP権利待ちレコードも現行ビルドの失敗理由として扱う。権限登録後は利用者が明示的に再実行する。`DOCUMENT_PORTAL_KIT` は外部ツールの契約On/Offだけを保持し、Astackの標準テンプレートや公開画面に制限を追加しない。

外部Webサーバーに配布済みの静的文書や取得済みファイルはAstackのAPIを経由しないため、製品側の状態変更だけでは取り消せない。公開先の既存の原子的公開能力・失敗時の制約は [ジョブ・公開](jobs-and-publication.md) と [ビルド計画](build-execution-plan.md) に従う。

## 管理画面

ADMINのシステム設定から `/settings/license` を開く。共通のページ見出し・カード・一覧・`FormModal` を使用する。

- 現在の状態、契約者、edition、有効期間、機能、利用数/上限、現在のインストールID、OS認識ホスト名、認証対象IPv4、登録履歴を表示する。
- 共通のライセンス詳細・プレビューで署名済みの「登録時のホスト名」を表示する。旧形式・自動体験版・未登録は「記録なし」。現在のOS値と混同せず、ホスト名そのものは翻訳しない。登録ホスト名は管理情報で、認証条件は引き続きインストールIDの照合とする。
- 「発行依頼情報をコピー」は現在のインストールIDとホスト名だけをコピーする。IPアドレスや契約情報をコピーせず、発行担当者への自動通知もしない。利用者が担当者へ伝える。Clipboard APIや代替コピーが失敗したら成功と表示せず、手動選択を案内する。
- ID未取得時は登録とコピーを無効化し、環境・保存領域の確認を案内する。旧方式で認証中でも、発行依頼には旧登録IDではなく表示中の新しいIDを使う。
- `null` の上限は無制限、`0` は利用枠なし。計数不能を0件に置き換えない。
- ファイル選択→サーバー検証→プレビュー→同じファイルの登録の順に操作する。ファイル変更でプレビューを破棄し、確定時にも再検証する。
- 登録中の二重操作・モーダルを閉じる操作を抑止する。通信断や503では保存結果を断定せず、状態の再取得と復旧を案内する。
- 初回取得失敗と、前回取得済みの表示を保持した再取得失敗を区別する。履歴取得の失敗を履歴0件として表示しない。
- 一般ユーザーのバナーは最小状態だけを表示する。契約者・ID・履歴を開示せず、ADMIN画面へ自動転送するループを作らない。

初回管理者のパスワード変更必須状態を優先する。まずパスワードを変更して再ログインし、ライセンスを登録する。

## 社内開発用共通ライセンス

[v6契約](../contracts/license-v6.md) の署名付き共通ライセンスは、ビルド時に許可した開発用鍵のみで受理する。インストールID・ホスト名の入力や環境再構築ごとの再発行は不要。有限期限・製品版範囲・機能・利用枠・署名検証を維持し、正式配布ビルドは拒否する。OSDCは同梱できない。対応する開発版では端末情報が取得不能でも共通ライセンスの登録を許可する。

## 端末に紐付くインストールID

新方式のIDは、現在のOS認識ホスト名・認証対象IPv4の集合・永続化した初回日時から導出する。保存したUUIDをそのまま認証に使わない。通常の再起動でこれらが同じならIDは変わらず、ホスト名変更、対象IPv4の追加・削除・変更、認証対象IPv4の集合が変わるNIC構成変更、初回日時の異なる再インストールではIDが変わる。その場合は管理画面のIDとホスト名で再発行する。ライセンスファイルや旧 `installation.id` だけを他環境へコピーしても、新方式の照合先は変わらない。

### OS情報とIPv4の選別

ホスト名はLinuxでは `/etc/hostname`、取得できない場合はOSのhostnameコマンド、WindowsではSystem32のhostname.exeから取得する。DNSの逆引き、Astackの表示用設定、`HOSTNAME` / `COMPUTERNAME` の値を認証情報にしない。前後空白を除き、小文字化し、末尾のドット1個を除去する。空、localhost等の汎用名、不正なホストラベルは取得不能とする。対応外OSも取得不能となる。

IPv4はOSのNIC一覧から次の条件で選別する。固定IPかDHCPかの区別はせず、DHCPで割り当てられたIPv4も含める。プライベートIPv4も対象となる。複数の対象IPv4は数値順で並べ、重複を除去するため、NICの列挙順だけではIDは変わらない。

- 起動中のNICを対象とし、loopback、virtual、point-to-pointのNICを除外する。
- Docker・veth・bridge・仮想ネットワーク・VPN・トンネル等は、OSの属性とNIC名/表示名の既知パターン（docker、veth、br-、virbr、vnet、vmnet、tun/tap/wg/utun、tailscale、zerotier等）で除外する。全製品の仮想NICを網羅する判別ではない。
- IPv6、未指定アドレス、loopback、link-local、multicast、全ビット1のbroadcastを除外する。
- ホスト名または対象IPv4が無い場合、一部NICの列挙・属性取得に失敗した場合は、部分的な情報でIDを確定しない。管理者画面には取得できた項目を表示しても、IDは未取得とする。

DHCP更新やNIC増設、VPN/仮想NICの導入前に、画面の認証対象IPv4とIDへの影響を確認する。同一環境での復元には、初回日時だけでなくホスト名と認証対象IPv4の集合も同じである必要がある。

### 導出と照合

正規化した文字列 `ASTACK-MACHINE-INSTALLATION-V1\n{hostName}\n{IPv4をカンマ連結}\n{初回日時}\n` のUTF-8をSHA-256でハッシュ化する。先頭128ビットのversionを8、variantをRFC形式へ設定し、小文字の標準UUID表記（UUIDv8）とする。初回日時は `installed-at` のUTC秒精度 `YYYY-MM-DDTHH:mm:ssZ`。新方式で登録した署名済みIDは、状態確認・業務操作でも現在のOS情報から再計算したIDと比較し、不一致・取得不能なら `INVALID` として新規更新を許可しない。

OSの情報を用いるオフラインの紐付けであり、ハードウェア固有IDや改変耐性のある端末証明ではない。管理権限でOS情報・保存データ・製品本体を複製/改変することの完全防止や、オンライン失効を保証しない。

### 初回日時と旧方式からの移行

`installed-at` が正常ならその値を維持する。無い場合に旧体験記録 `automatic-trial.started` / `automatic-trial.json` があれば、保存済み `startedAt` を引き継ぐ。両方がある場合は日時一致が必要。記録破損や不一致を現在時刻で置き換えない。

署名検証済みの既存UUIDv4ライセンスが旧 `installation.id` と一致する場合に限り、経過措置として旧方式の認証を継続する。初回日時も旧体験日時も無い旧正式ライセンスでは、署名検証済み `issuedAt` を移行用の初回日時へ採用する。元の初回日時を復元したことを意味しない。正式ファイルまたは `formal-license.seen` があるのに復元根拠が無ければ、新しい現在時刻を採用せず復旧を要求する。

旧方式ではIDファイルを含む複製を端末情報で区別できない限界が残る。新方式への移行は、管理画面に表示する新IDで再発行する。新規アップロードの検証・登録は常に現在の新IDとの一致を要求するため、旧UUIDv4ファイルの再アップロードによる継続登録はできない。新方式へ切り替えた後は現在の端末情報が必須となる。UUIDの方式と署名envelopeの版は別の概念で、ID導出方式自体は署名payloadの項目追加を必要としない。v5のホスト名追加は管理情報を記録する別の変更である。

初回日時と自動体験期間は別の管理要件である。自動体験は既存の開始・終了日時、権利、枠、記録照合を維持し、ID変更・正式ライセンスの再発行・移行で30日間を再開始しない。詳細は [体験版の初回日時と再インストール](trial-and-osdc-bundling.md#初回日時と再インストール) を参照する。

## API

応答は `Cache-Control: no-store`。ADMIN操作はURL認可に加えControllerと製品サービスのメソッド認可でも保護する。

| API | 権限・用途 |
| --- | --- |
| `GET /api/license/summary` | 認証済み。state、allowsUpdates、featuresのみ |
| `GET /api/license/status` | ADMIN。詳細、現在のインストールID、端末情報、利用数、超過有無、信頼鍵配備有無 |
| `POST /api/license/validate` | ADMIN。multipartのfileを検証し、プレビューを返す。保存しない |
| `PUT /api/license` | ADMIN。同じfileを再検証して登録/更新。操作者は認証主体から確定 |
| `GET /api/license/events?page=0&size=25` | ADMIN。content/totalElements/totalPages/number。pageは0以上、sizeは1〜100 |

詳細状態とプレビューのトップレベル `hostName` は署名済み登録ホスト名（v5）で、旧形式・自動体験版・未登録はnull。環境不一致でINVALIDになった場合も署名検証済みの登録値を表示する。現在のOSホスト名は `machineIdentity.hostName` として別に返し、一般ユーザーのsummaryにはどちらも追加しない。

詳細状態の `installationId` は常に発行依頼用の現在導出UUIDで、取得不能時は空文字。登録ファイルのIDと不一致の場合も登録IDへ差し替えない。`machineIdentity` はADMIN向けの次の情報だけを返し、一般ユーザーのsummaryへ追加しない。

| 項目 | 意味 |
| --- | --- |
| `hostName` / `ipAddresses` | 正規化したOS認識ホスト名と認証対象IPv4一覧 |
| `available` | 現在の端末情報・初回日時からIDを導出可能か |
| `bound` | 検証済み登録ファイルが新方式UUIDv8か。一致判定やACTIVE状態そのものではない |
| `legacyLicense` | 検証済み登録ファイルが旧方式UUIDv4か |
| `registeredInstallationId` | 検証済み登録ファイルのID。登録情報が無ければnull |

アップロードは64KiB以下。状態は `UNLICENSED` / `ACTIVE` / `NOT_YET_VALID` / `EXPIRED` / `INVALID`。署名が正しい期限切れ・開始前ファイルはプレビューできるが、登録するのは現在ACTIVEなファイルだけとする。

未認証401、認可不足403、不正ファイル400。業務制限は403と `LICENSE_REQUIRED` / `LICENSE_EXPIRED` / `LICENSE_NOT_YET_VALID` / `LICENSE_INVALID` / `LICENSE_FEATURE_UNAVAILABLE` / `LICENSE_LIMIT_EXCEEDED`。計数不能は503 `LICENSE_USAGE_UNAVAILABLE`、保存・監査の障害は503 `LICENSE_STORAGE_UNAVAILABLE`。JWTのパスワード変更必須の拒否をライセンスのエラーに置き換えない。

## 件数と同時実行

| 対象 | 計数と増加経路 |
| --- | --- |
| プロジェクト | 全プロジェクト。通常作成・プロジェクトインポート。プロジェクトにはゴミ箱状態がないため全行を数える |
| ライブラリ | ゴミ箱以外を全プロジェクト合算。通常作成、プロジェクト/ライブラリ取り込み、復元 |
| 文書 | 有効なライブラリ別DBの `build_file.deleted=false` の元文書。複数出力・ページ・DocumentMeta・過去成果物を加算しない。別ライブラリの同じ原本は別件 |

v1は引き続き旧署名形式で検証するが、旧ユーザー枠をプロジェクト枠へ読み替えない。旧v1のプロジェクト枠は無制限で、ライブラリ/文書枠は維持する。旧ユーザー枠は適用しない。APIの上限値はnullまたは10進文字列で返し、64bit上限の画面表示を丸めない。

上限は非負整数またはnull。正規の上限引下げは登録できるがデータを自動削除せず、`overLimit` を表示して増加操作を拒否する。

プロジェクト/ライブラリの増加操作はメインDBの `license_quota_lock` の1行をJPAで排他し、同じJPA接続でflush後の件数を読む。同一外側トランザクション内の複数作成や復元でも自分の未コミット変更を含める。取り込みは必要件数を一括判定し、失敗時に部分作成しない。ゴミ箱からの復元はそのライブラリの文書も加算する。

文書は `license_document_reservation(library_id, relative_path)` にメインDBの独立トランザクションで先に予約する。既存パスと未確定予約の和集合を数え、同じパスを二重計上しない。文書登録の成功コミット後だけ予約を消去し、失敗/再起動では枠を残す。同じパスの再実行で予約を再利用・確定できる。時間経過だけでの予約削除は行わない。

ライブラリDBの `license_document_write_lock` を使って登録を直列化し、単件・バッチ・並行処理で同じ有効パスの重複行を作らない。既存の重複データを自動削除せず、衝突する更新は拒否する。登録処理は製品既存のトランザクション管理へ参加し、外側JPAロールバックでlibraryDB行を早期コミットしない。

利用数は各有効ライブラリの実DBと未確定予約から集計する。読み取りに失敗したら利用数不明とし、増加操作を許可しない。この方式は利用数照会・増加判定で複数DBを読むため、多数ライブラリでの性能は導入前に評価する。予約を手作業で消す場合は実登録とジョブ状態の照合が必要であり、容量不足の解消だけを目的に一括削除しない。

## 保存・監査・中断復旧

保存先は `astack.license.directory`（既定 `./data/license`）。管理者だけが書き換えられるデータ領域に置く。

| 保存物 | 意味 |
| --- | --- |
| `installed-at` | 新方式IDの導出に使うUTC秒精度の初回日時。更新時に削除・再生成しない |
| `machine-identity.lock` | 初回日時の読取り・移行・生成のプロセス間排他 |
| `installation.id` | 旧方式UUIDv4。既存正式ライセンスの経過措置と自動体験記録の照合に使用。新方式の発行依頼IDではない |
| `installation.lock` | 旧ID初回生成のプロセス間排他 |
| `automatic-trial.started` / `automatic-trial.json` | 自動体験の開始・期限・権利・枠の対応する保存記録 |
| `formal-license.seen` | 正式ライセンス利用済みの記録。体験版への復帰を防ぐ |
| `license.lic` | 現在の署名済みファイル |
| `license.previous-{eventId}.lic` | 更新中の旧ファイル退避。正常確定後は削除 |
| メインDB `license_events` | 操作者・UTC時刻・結果・検証済みID/keyId・新旧ファイルのSHA-256 |

登録は①検証、②PENDING履歴の永続化、③旧ファイルの原子的退避、④新ファイルの原子的保存、⑤ACTIVATED履歴の確定の順。履歴確定前は新しい権利を業務へ公開しない。失敗時は旧ファイルを戻してFAILEDを記録する。ロールバックも失敗した場合はINVALIDとして業務処理を拒否し、成功応答を返さない。不正な登録試行は入力本文を保存せずREJECTEDとして記録する。

再起動時にPENDINGを新旧ハッシュ・退避ファイルと照合する。新ファイルと一致すればRECOVERED、旧ファイルと一致すればFAILED。旧ファイルの退避直後に中断した場合は退避を戻す。不明な内容を黙って正規化せず、復旧が必要な状態として表示する。

64KiBを超える破損した既存ファイルも、現在IDを取得できれば新しい正規ファイルで置換できる。旧ファイルはメモリ全読込みせず退避する。新方式では旧 `installation.id` の欠損・改変を現在IDの照合に使用しない。旧方式継続認証と自動体験には元の旧IDが必要なため、一式の復元対象とする。

初回日時の消失・破損や端末情報の取得不能はアプリ全体を落とさず、ID未取得・無効状態として復旧を案内する。既存正式ファイルがある状態で現在時刻を新しい初回日時として採用しない。元の日時記録をバックアップから戻し、環境を確認した後、状態を再取得する。旧正式ライセンスからの日時移行だけは前述の署名済み発行日時を利用する。

バックアップはライセンス保存先一式（初回日時、旧ID、正式ライセンス、体験記録、正式利用済み記録、更新中の退避ファイル）とメインDBを対応付けて取得する。復元前にサービスを停止し、同じ世代を戻す。同じ初回日時・OS認識ホスト名・認証対象IPv4の集合であれば新方式IDを維持できる。異なる環境への復元・移設では再発行する。対象NICが変わらなくてもDHCPのアドレス変更でIDが変わることに注意する。OSの親ディレクトリ耐障害性や公開先の原子性の制約は残る。

## 配備前の準備

製品の信頼鍵は [ProductLicenseKeys.java](../../backend/src/main/java/jp/astack/backend/license/ProductLicenseKeys.java) の `PUBLIC_KEYS` 定数へ `keyId: Ed25519 SPKI PEM` としてビルド時に組み込む。旧classpathリソース `license/trusted-keys.json` は廃止し、外部ファイル・環境変数・システムプロパティ・アップロードから任意の鍵を追加しない。リリースでは発行システムから取得した公開鍵JSONと、組み込むkeyId、別途承認したSPKI指紋を [build/build-jar.sh](../../build/build-jar.sh) に指定する。選択鍵の形式・指紋を確認し、通常ソースを変更せず一時ビルドソースへ反映し、JAR内の鍵を検査してから配布先へコピーする。JSONファイル自体はJARに含めない。鍵の更新はレビュー・製品再ビルド・更新で行う。通常ソースのマップは空で、製品用鍵も旧ブランチの露出鍵も含まない。**正式な発行鍵と対応する公開鍵を組み込んだJARを配備するまで、正規ライセンスを登録できない。** 画面では発行準備が必要な状態を案内する。テスト鍵を製品へ流用しない。ファイル編集だけでの信頼鍵差し替えを防ぐ簡易対策であり、本体の解析・改変を完全に阻止する仕組みではない。

開発専用の夜間配布にも、承認した開発用公開鍵・keyId・指紋を必須とする。[Windowsナイトリービルドの運用](../nightly-windows-build.md#開発用ライセンス公開鍵)を参照する。鍵なしJARを配布せず、起動時の信頼鍵追加も行わない。通常ソースと正式配布の信頼鍵は変更しない。

初期導入と既存環境への制限有効化は、発行側の構築・鍵配備・必要ライセンスの発行・登録手順・復旧手順の確認後に行う。先にコードだけを稼働環境へ適用しない。本変更はマージや本番適用の承認を代替しない。

配布JARは [build/build-jar.sh](../../build/build-jar.sh) のようにフロントエンドのビルドを先に行う経路で作成する。backendだけのMavenビルドでは管理画面の静的配信物を更新しない。検証用のVite出力や生成済みJavaScriptを手作業でソースへ混ぜず、配布時にライセンス画面と登録経路の存在を確認する。

新規メインDB表は `license_quota_lock`、`license_document_reservation`、`license_events`。既存の初期化方針に合わせて生成するが、`ddl-auto=validate` やDDLを許可しない運用ではJPA検証/サービス起動より前に用意する。

```sql
CREATE TABLE IF NOT EXISTS license_quota_lock (id BIGINT PRIMARY KEY);
INSERT INTO license_quota_lock(id) SELECT 1 WHERE NOT EXISTS (SELECT 1 FROM license_quota_lock WHERE id=1);
CREATE TABLE IF NOT EXISTS license_document_reservation (
  library_id BIGINT NOT NULL, relative_path VARCHAR(4000) NOT NULL,
  PRIMARY KEY(library_id, relative_path));
CREATE TABLE IF NOT EXISTS license_events (
  id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  occurred_at VARCHAR(30) NOT NULL, actor VARCHAR(255) NOT NULL, result VARCHAR(30) NOT NULL,
  license_id VARCHAR(100), key_id VARCHAR(100), content_hash VARCHAR(64), previous_hash VARCHAR(64));
```

ライブラリDBの接続初期化は `license_document_write_lock(id BIGINT PRIMARY KEY)` とid=1を追加する。独自のDDL禁止環境では各DBにも同じ準備が必要。既存データを移し替えず、DB・ファイル・権利を自動補完しない。

## 根拠と検証

- [端末情報](../../backend/src/main/java/jp/astack/backend/license/MachineEnvironment.java)、[ID導出・初回日時](../../backend/src/main/java/jp/astack/backend/license/MachineInstallationIdentity.java)、[署名ファイルとID照合](../../backend/src/main/java/jp/astack/backend/license/LicenseStore.java)
- [製品サービス](../../backend/src/main/java/jp/astack/backend/license/ProductLicenseService.java)、[監査](../../backend/src/main/java/jp/astack/backend/license/LicenseAuditJournal.java)、[API](../../backend/src/main/java/jp/astack/backend/controller/LicenseController.java)
- [業務ポリシー](../../backend/src/main/java/jp/astack/backend/license/LicensePolicy.java)、[サービス入口・定期処理の適用](../../backend/src/main/java/jp/astack/backend/license/LicenseOperationAspect.java)、[件数制限](../../backend/src/main/java/jp/astack/backend/license/LicenseQuotaService.java)
- [ファイル形式](../contracts/license-v1.md)、[画面](../../frontend/src/pages/LicenseManagementPage.tsx)、[UI方針](../ui-design-guidelines.md)
- MVC権限/実JWT・実署名から実API/DBまでの登録・改ざん・期限・上限・更新、実H2での同時枠取得・JPA/JDBC排他・ロールバック・予約保持、ファイル/監査中断・ID復旧を自動試験する。
- フロントエンドは型・Lint・Node試験に加え、APIモックでPC/スマートフォンのプレビュー・二重操作・再選択・失敗・権限・復旧案内を検証する。実発行サービス/正式鍵とのE2E、全公開先の転送中失効、数百以上のライブラリ性能は本段階の確認対象外。
