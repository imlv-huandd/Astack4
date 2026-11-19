# ビルド変更計画・差分公開・成果物保持の設計

- 対応: [#6680](https://developer.astack.jp/redmine/issues/6680)
- 設計日: 2026-10-24（日本時間）
- 対象: FULL / DIFF / TEMPLATE、内部公開・外部ストレージ公開
- 調査の原記録: [ビルド処理全体の調査](../reviews/build-pipeline-review-20261024.md)。同書の旧コードに対する再現結果は履歴として保持する。
- 本書はコード上の実装契約を記す。統合テスト結果・稼働適用結果は主担当が確定後に追記する。未検証事項を成功とは扱わない。

## 1. 処理順序と責任

```mermaid
flowchart LR
    A[予約・世代と設定の固定] --> B[全走査・公開設定とACL差分]
    B --> C[変更計画を保存]
    C --> D[必要変換・検索更新・HTML生成]
    D --> E[公開設定差分と欠落を補完]
    E --> F[公開manifest・未完了操作を保存]
    F --> G[差分転送・削除 / STAGING切替]
    G --> H[公開参照・実行計画・日時を確定]
    H --> I[参照保護と猶予付きGC]
```

原本の追加・更新・削除、フォルダの追加・削除に加え、ACL、公開期間・公開除外、表示名等のメタ設定、取得方式、テンプレート・ビューア、検索/AI情報、失敗処理・欠落成果物を独立した差分として扱う。公開結果を変える仕事も未完了操作もないDIFFは公開と公開日時更新を省略し、実行履歴を残す。

新しい計画は既存のイベント駆動・変換ジョブを置き換える全機能スケジューラーではない。永続した走査根拠・タスク進捗と公開操作を既存処理に接続する段階移行である。

## 2. 計画と世代

`BuildPipelineOrchestrator` が `BuildExecutionPlanService.start(libraryId, buildId, mode)` を呼ぶ。ライブラリDBに次のテーブルを `CREATE TABLE IF NOT EXISTS` で追加する。既存文書・フォルダIDは変更しない。

| テーブル | 主キー・主要列 | 用途 |
| --- | --- | --- |
| `build_execution_plan` | `build_id`, `mode`, `state`, `snapshot_json`, `error_message`, 作成・更新日時 | 世代と走査スナップショット |
| `build_execution_task` | `(build_id, task_key)`, `kind`, `target_id`, `target_path`, `revision`, `state` | 対象別の必要作業と状態 |

`CrawlerService` は変換ジョブ作成直前に `recordScan` を呼ぶ。原本サイズ・mtime・内容ハッシュ、変換指紋、ACL、変換状態を保存し、`CONVERT` / `GENERATE` / `DELETE` / `DELETE_FOLDER` タスクを登録する。公開前にも記録・同期し、`PUBLISH` を追加する。タスクは既に登録済みなら重複作成しない。

計画の状態は `SCANNING → PROCESSING → PUBLISHING → COMMITTED`、変更なし成功は `UNCHANGED`。失敗・取消は `FAILED` / `CANCELLED`、後の正常な走査で置き換えられた未完了世代は `SUPERSEDED`。タスクは既存DBの変換状態・フォルダ生成状態と同期する。

走査開始時には変換dispatchを停止し、計画保存・ジョブ登録と走査の成功後にだけ有効化する。古いdispatcherが取得したジョブ一覧も取得時のbuildIdで照合する。

`BuildContext` の `beginOperation` / `endOperation` は非同期変換・HTML生成の実行リースを数える。取消で現在世代を無効化しても実処理のリースが残る間は新世代を開始しない。DB更新・イベント確定前にも対象 `buildId` を照合する。待機中予約の取消と実行中ビルドの停止は別の対象として扱う。公開確定と取消は同一の世代ロックで順序を確定し、公開確定後の取消は競合として拒否する。同期検索登録の失敗・部分失敗は変換失敗へ伝播し、次回走査で再試行する。検索・ベクトル削除とACL更新もHTTP成功だけでは完了とせず、タイムアウト・部分失敗・競合を確認する。

遅延した終了ログはlibraryIdだけでなくbuildIdに結び付け、新ジョブのログを閉じない。同秒開始でのファイル名衝突は既存ログを上書きせず連番で回避する。

## 3. 公開可否・ACL・再公開

`DocumentMetaService.resolveEffective` と `resolveEffectiveFromCache` が共通の実効設定を返す。`PublicationPolicyService` は祖先FOLDER（ルート含む）の公開除外と期間を合成する。除外は論理和、開始日は最大、終了日は最小とし、子の設定で親制限を解除しない。公開可否は既存DTOの `isPublished()` で判定する。

`CrawlerService` はWAITING以外も含む全生存文書を検査する。非公開への遷移では検索データ・ベクトル・変換ジョブを取り下げてから公開理由付き `SKIPPED` を保存する。公開可能に戻ると `WAITING` に戻し、変換・検索を再構築する。親一覧も更新対象にする。

`CrawlerAclRefreshService.refresh(library, provider)` は内容・mtimeに依存せずファイル/フォルダACLを取得する。既存フォルダのACLを登録処理で先に上書きせず、専用処理で差分を比較する。権限変更は検索ACLと一覧生成へ反映する。索引更新より先に一覧の再生成要求を保存し、検索側の失敗でも要求を失わない。ACL取得不能は許可へフォールバックしない。

LOCAL/CIFSでは読取データに適用されるALLOW/DENY ACEを `A:識別子` / `D:識別子` の順序付き形式で保存する。owner一致だけでは閲覧を許可しない。旧形式の識別子だけの値は互換性のため明示許可として読むため、保護ライブラリの導入後走査で新形式へ更新する。これはOSの全アクセス権を代替する一般的なNTFS実効アクセスエンジンではなく、公開閲覧に必要な読取権の評価である。

`PortalController` はHTML、PDF、SVG、マニフェスト、コピー原本で公開可否を都度検査する。原本直送は `PortalDownloadController` でも同じ実効設定を使用する。DBへ対応しない古い成果物・削除済み文書/祖先フォルダは拒否する。テーマ/QRの既存共通資産例外を維持する。内部フォルダ一覧は配信時に描画し、公開不可項目を一覧・ツリーから除く。描画で公開日時やディスク上のHTMLは変更しない。

生成側も同じ公開判定で対象を絞り、設定解決失敗時は処理を失敗させる。検索側は公開可否・ACLで文書ID集合を絞り、返却前にも現在状態を確認する。

## 4. 設定だけの変更と公開取り下げ

`DownloadPublicationService.prepare(library, buildId, buildPath)` は前回成功時の状態と実効設定のSHA-256指紋を比較する。文書の取得方法、公開先ディレクトリ、公開可否、表示名・期間・コピー等の実効DTOを比較対象にする。フォルダ設定も独立して比較する。

返却する `Plan` は `state`, `removeOrigins`, `removeDirectories`, `removeFiles`, `changed` を持つ。メタ情報だけが変わっても親一覧と文書HTMLを生成する。非公開文書はプレビューと原本を含む公開ディレクトリ全体、非公開フォルダは配下を削除する。ルート非公開時は公開先ルート自体を削除せず `index.html` を削除する。flat公開の文書成果物は文書単位で削除する。

ステージ済み出力からも公開禁止成果物を除去する。コピー原本の新規準備は取得方法・出力ディレクトリ変更時に限り、表示名だけの変更で元ストレージを再取得しない。`commit` は公開成功時だけ状態ファイルをatomic move（非対応時はreplace move）で更新し、失敗時は同じ差分を再試行する。

## 5. 公開manifestと差分転送

`PublicationManifestStorage.plan` は生成ファイルのサイズとSHA-256を計算し、再試行用の内容を非公開 `objects/<hash>` に保存する。公開先の識別は `StorageProvider.describeLocation() + "|" + destination` から作る。未完了の転送候補・ディレクトリ削除・単一ファイル削除・旧形式出力のクリアを `pending.properties` へ永続化してから公開操作へ進む。

`uploadChanged` は前回署名と公開先の存在を照合し、同じファイルの転送を省略する。欠落公開ファイルは保存済みobjectから復元する。`applyDeletes` は単一ファイルとディレクトリを区別し、存在しなければ成功扱いとする。同パスへ文書が再作成された場合は今回生成した成果物を古い削除要求より優先する。

FULLのSTAGINGでは既存 `publicationSupport().publishStaged` による切替を維持する。DIRECTおよびDIFFのマージ公開は署名差分の転送・削除を実行する。**DIRECTの全ファイル転送と削除を一括で原子的に見せることは保証しない。** 中断時には一部新旧ファイルが混在し得る。pendingの再試行可能性と、公開完了を誤って確定しないことを保証する設計である。

## 6. 公開確定と保存場所

`PublisherService` はリモート操作成功後に取得方法状態を確定し、削除フォルダの処理済み化・削除済み文書のDB purge・公開manifestの確定を行う。その成功結果を受けて `BuildPipelineOrchestrator` が `PublishedFileStorage.commitPublishedBuild`、公開日時、実行計画の完了を確定する。

| 場所（`<library-db-path>` 配下） | 内容 |
| --- | --- |
| `<libraryId>/download-publication.properties` | 最後に公開成功した実効設定・取得方法指紋 |
| `publication-manifests/<libraryId>/<destinationHash>/current.properties` | 公開済みのファイル署名・object参照 |
| 同 `previous.properties` / `pending.properties` / `objects/` | 前回参照、未完了操作と転送候補、再試行用blob |
| `published-files/<libraryId>/current.properties` / `previous.properties` | 公開済み文書版・対応原本の現行/一世代前参照 |
| 同 `versions/<revision>/` / `originals/<revision>/` | 文書版と原本の非公開保存 |
| `artifact-cache/<libraryId>/` | 変換成果物のキャッシュ |

各ファイルのatomic move、DB更新、リモート切替は一つの分散トランザクションではない。リモート成功後のローカル確定失敗では、次回に冪等に再実行する。公開版と対応原本の保持により、変換失敗時の旧公開版と最新版原本を混同しない。

## 7. 移行と再起動後の復旧

既存データは消去せず、計画テーブルと状態ファイルを追加する。公開署名がない最初の実行では設定差分・出力署名を初期化するため再生成・転送が発生し得る。既存公開版は `ensurePublishedSnapshots` で実際の公開先から取り込み、原本・プレビューの対応が不明な版を最新原本として扱わない。

再起動時のRUNNING予約は `BuildWorkerPoller.markRunningJobsFailedOnStartup` がFAILEDへ変更する。途中の外部変換をそのまま再接続して続行する方式ではない。次のビルドで原本・現在設定を再走査し、古い未完了計画を `SUPERSEDED` にする。公開manifestのpendingとblobは残るため、転送・削除は次回に再試行できる。自動的に全FAILED予約を再投入することは本機能の保証に含めない。

状態ファイル・ライブラリDB・private成果物をまとめてバックアップする。公開先だけを復元しても計画・署名との対応は復元されない。切戻し時は新しい状態を旧版が理解することを前提にせず、主担当の稼働手順に従う。

## 8. 安全GC

`ArtifactRetentionService.collect(libraryId, buildId, dryRun, retryBuildIds)` を変更のある公開成功後に呼ぶ。無変更終了時はGCも省略する。設定は `build.retention.enabled=true`、`grace-days=7`、`dry-run=false` が既定。ただし初回は必ず候補計測のみとし、`gc-inspected.properties` を残す。以後もdry-run設定が有効なら削除しない。

現行・一世代前公開参照、DBのoutputPath、現在ビルド、再試行可能計画のbuildId、公開manifestのcurrent/previous/pendingが参照するobjectを保護する。未参照で猶予日数を過ぎたprivate版・原本・キャッシュ・objectだけを候補とする。参照の読込不正、DB照合失敗、領域逸脱、シンボリックリンク検出時は削除を中止する。元ストレージ・公開先はGC対象外。

`gc-report.properties` に候補件数・容量・削除件数・保護件数を保存する。GC失敗で確定済み公開を失敗へ戻さず、次回成功時へ延期する。artifact-cacheとpublished-filesの物理的な重複保存を完全統合する改修は含めない。

## 9. 現在の制約と残課題

- 外部静的公開の公開期間変更は次のビルド成功時に反映する。期限到来専用の即時実行スケジューラーや外部CDNキャッシュ失効は未実装。内部配信はアクセス時に拒否できる。取得済みコピーは回収できない。
- 旧形式ACLは読取マスクを復元できないため、保護ライブラリでは導入後走査が必要。実NTFSの代表的なALLOW/DENY/継承条件は実環境検証の対象に残る。
- 同一フォルダ内の全HTML生成、フォルダツリーの組立、メタ・要約照合のすべてを文書dirty集合と一括取得へ置き換えたわけではない。転送量削減と生成CPU/DB往復削減は別に計測する。
- 無変更判定でも原本走査・署名参照・公開先存在確認等の読取処理は必要。上記の「何もしない」は公開内容と公開日時への副作用を省略する意味である。
- テンプレートZIPの実展開量制限、resources.dirの境界検証、SSEのURL JWT・ログ漏洩は別の未対応事項として調査書の対応表で追跡する。

## 10. 検証記録

- 実装: `1cc460a`（#6680、[PR #13](https://github.com/nt-bsys-dev/Astack4/pull/13)）。mainへのマージは別途レビュー対象。
- [実装最終CI](https://github.com/nt-bsys-dev/Astack4/actions/runs/40058114812): 1589件、失敗0、エラー0、skip13。Checkstyle成功。対象SDKを使用した部分失敗/競合/timeoutの追加回帰を含む。
- 2026-10-24 12:28 JSTに稼働環境へ適用。停止中にH2 DB10個と旧起動設定を退避。Java変更前クラス34件の逆アセンブル一致を確認し、稼働jarの94クラスだけ差し替え、既存画面・テンプレート・QR資産を保持。起動後health=UP、反映直後の既存設定と索引件数は不変。
- 専用新規ライブラリのビルド#557〜#571: 15件成功、47項目のassert成功。空ビルド、無変更、ファイル追加/更新/削除、フォルダ追加/削除、文書/フォルダ公開停止と再公開、欠落PDF1ファイルだけの復元、GCを確認。
- 公開停止の設定後、ビルド前にも旧プレビュー・コピー原本・フォルダindexへのHTTP要求が拒否されることを確認。
- 専用ライブラリDB: 計画15件（COMMITTED 12 / UNCHANGED 3）、タスク40件すべてSUCCEEDED。削除文書のDB行は0件、削除フォルダはDELETED/GENERATEDとなり次回へ持ち越さない。
- GC: 専用の14日前の未参照ダミー1件は初回dry-runで保持し、その後の変更あり公開成功で削除。古くした現行/前回の参照版とDB参照キャッシュは保護。無変更時にはGCレポートも不変。
- 既存ライブラリは初回整合ビルド#572成功後、無変更DIFF #573/#574で転送0・公開日時維持。公開959ファイルの全内容ハッシュとmtimeが2回とも不変。公開ページ・管理画面・QR共通資産はHTTP 200。
- 既存ライブラリの初回GCはdry-run、候補0/削除0。未参照でも7日猶予に達しないprivate成果物を即座に削除しない。

運用証拠（DBバックアップ・差替えjar/ソース指紋・CI・実測JSON）は稼働ホストの `build-pipeline-release/20261024-122438` に保存した。旧版への切戻し用起動設定も同梱。検証専用project/library/storageのIDを台帳で照合して片付け、API仕様上保持される履歴は残す。実機NTFSの全条件、全ストレージの実接続障害、プロセスクラッシュの全タイミングを網羅した故障注入は未実施。対応する単体・統合テストと、本書のDIRECT等の制約を適用範囲とする。

## 実行モード指定の一本化（#6725）

現行の利用者向けモード指定はジョブ・スケジュールの `mode` のみとする。ライブラリの `fileDiff`・`allContentsOnNext`・`buildAllEverytime` は保存互換用の旧項目であり、実行や次回消費には使わない。手動・スケジュール・再実行・失敗・複数予約を含む扱いは [ジョブ・公開のモードと処理経路](jobs-and-publication.md#モードと処理経路) を参照。

上記のモード指定への一本化は、ライブラリ設定の実処理が未提供である間の暫定対応である。#6725の機能実装・検証は今後実施し、それまでチケットをクローズしない。将来の優先関係・次回全件の消費時点・失敗時の扱いは、その実装時に定義する。
