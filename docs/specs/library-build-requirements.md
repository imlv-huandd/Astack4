# ライブラリ設定とビルドの対応表

関連チケット: [#6759](https://developer.astack.jp/redmine/issues/6759)

ライブラリの初期登録・設定保存が成功した後、未反映の設定があれば、必要なビルドと理由を一つの確認ダイアログに表示する。「あとで実行」で閉じても、一覧の「フルビルド待ち」「差分更新待ち」「テンプレートビルド待ち」から再度確認・予約できる。保存失敗時には確認を表示しない。

## ビルド対応表

ここでいう「不要」は、公開成果物への設定反映を目的としたビルド予約が不要であることを示す。原本の追加・更新、専用AI処理、外部設定の変更によって必要となるジョブは別途実行する。

| 設定項目 | フィールド | 保存後に案内する処理 | 理由・反映範囲 |
| --- | --- | --- | --- |
| 初期登録・一度も成功していないライブラリ | 初期ビルド状態 | フルビルド | 走査、変換、ページ生成、索引、公開を初期化する |
| ライブラリ名 | `name` | フルビルド | 内部公開URLや出力先への影響を含む |
| タイトル、テンプレート選択 | `title`, `template` | テンプレートビルド | 全フォルダの公開HTMLを再生成する |
| 入力元接続、入力パス | `pathConfig.sourceConnectionId`, `sourcePath` | フルビルド | 入力元を再走査し、既存ファイルも再処理する |
| 公開先接続、公開パス、内部Webサーバー、サイトURL | `pathConfig.destConnectionId`, `destPath`, `useInternalWebServer`, `siteURL` | フルビルド | 公開先・出力URLを含めて再構築する |
| パス・文字列による除外条件 | `pathConfig.exclude`, `stringExclude`, `buildConfig.excludeString` | 差分更新 | 再走査し、追加対象・除外対象とページを更新する |
| メニューURL、APIパス | `pathConfig.menuURL`, `apiPath` | テンプレートビルド | 生成済みページのリンク先を更新する |
| API接続元制限・ユーザーヘッダー | `pathConfig.apiAccessRestricted`, `apiAllowedPeers`, `apiIdentityHeaderEnabled`, `apiUserHeader` | 不要 | API側で参照される実行時の制限設定 |
| 公開方法 | `pathConfig.publishStrategy` | 不要 | 次回公開時のSTAGING/DIRECT選択。設定変更だけでは公開内容を変えない |
| 出力構成・コンテンツ名 | `buildConfig.outputMode`, `enableContentsNameHash`, `libContentsItemNameLength` | フルビルド | 成果物の構成・名前が変わる |
| ショートリンク、QRコード、アクセスログ | `buildConfig.fileShortLink`, `fileQRCodeAccess`, `usingAccessLog` | テンプレートビルド | 公開ページ・共通アセットを更新する |
| 開始・終了コマンド、終了コード判定 | `buildConfig.startCommand`, `checkStartCode`, `startCode`, `continueOnStartCodeMatch`, `endCommand` | 不要 | 次回ビルド実行時の動作設定 |
| 差分判定期間・実行方式 | `buildConfig.fileDiff`, `fileDiffDuration`, `enableOndemandBuild`, `allContentsOnNext`, `buildAllEverytime` | 不要 | 公開内容の即時再生成対象にはしない。下記の既存実行モード制約を参照 |
| ファイルレジストリ | `buildConfig.fileRegistry` | 不要 | 現行の提供状況を変えず、ビルドの案内対象にはしない |
| ビルドログ | `buildConfig.logLevel`, `includeConverterOutput` | 不要 | 次回実行のログ出力に適用する |
| 検索・索引設定全般 | `searchConfig.*` | フルビルド | 変更のない原本も含めて索引と公開ページを再構築する |
| AIの有効化、モデル、入力元、生成要件、ログ | `aiProcessingConfig.*` | 不要 | 専用AIジョブで処理する。公開ビルドの案内がAI生成完了を保証するものではない |
| セキュリティ・原本公開設定全般 | `securityConfig.*` | フルビルド | 変換物、ダウンロード、公開HTMLの各段階への影響を包含する |
| 変換プロファイル選択、パラメータ上書き値 | `conversionProfileId`, `parameterOverrides` | フルビルド | 既存ファイルを新しい条件で再変換する。上書き理由・レコードID・順序だけの変更は対象外 |
| 変換形式・その他の変換設定 | `converterConfig.*`（次行を除く） | フルビルド | 再変換が必要な設定を安全側で扱う |
| 変換・ビューアタイムアウト、キャッシュ運用 | `converterConfig.viewerTimeout`, `converterTimeout`, `enableConvertCache`, `cacheRetentionDays` | 不要 | 次回処理時に参照する運用設定 |
| プレビュー形式 | `viewerConfig.format` | フルビルド | PDF/SVG等の既存変換物を再生成する |
| ビューア選択・固定バージョン | `viewerConfig.viewerId`, `viewerVersion` | テンプレートビルド | ページとビューア資産を更新する |
| 変換失敗時の公開方針、未対応ファイルの公開と拡張子 | `documentConfig.conversionFailurePolicy`, `publishUnsupportedFiles`, `unsupportedFilePublicExtensions` | 差分更新 | 最新の公開方針を既存ファイルにも適用する |
| ビルド前バックアップ | `documentConfig.enableBackUpLastBuild` | 不要 | 次回ビルド時の動作設定 |
| ファイル・フォルダのソート設定 | `displayConfig.*` | テンプレートビルド | 一覧HTMLを再生成する |
| 拡張設定、表にない将来の永続設定 | `extensionParams.*` 等 | フルビルド | 設定追加時の案内漏れを防ぐ。新設時に専用の分類も検討する |
| 最終ビルド日時、テンプレート署名、ビューア履歴、反映状態 | `buildConfig.lastBuildAt`, `lastTemplateId`, `lastTemplateSignature`, `lastViewerSelection`, `buildReflectionState` | 不要 | サーバー管理の成功履歴。判定の入力差分から除外する |

無効・非表示・未提供の設定を、この機能で新たに有効化することはない。ライブラリの差分／次回全件／毎回全件設定を実行モードへ反映する既存課題 [#6725](https://developer.astack.jp/redmine/issues/6725) は別管理であり、本機能の確認からは明示的に選ばれた `FULL` / `DIFF` / `TEMPLATE` を予約する。

## 複数変更と未反映状態

処理の包含関係は **FULL > DIFF > TEMPLATE** とする。FULLは走査・再変換・生成・公開を含み、DIFFは走査・必要な変換・生成・公開を含む。TEMPLATE相当の未反映変更があれば、DIFF/FULLでも公開前に全フォルダHTMLを再生成する。したがって、タイトルと除外条件を同時変更した場合はDIFFを一つ案内し、別途TEMPLATEを重ねて予約しない。

設定JSONの `buildReflectionState` に、項目ごとの最終成功値のSHA-256指紋と初期ビルド要否を保存する。認証情報や設定値そのものの履歴は追加しない。任意項目の未指定と空文字は同一とみなす。現在の設定と比較するため、見送り後に無関係な設定を保存しても要求を保持し、ビルド前に元の値へ戻した場合は不要な要求を解消できる。APIの `buildRequirement.configVersion` は判定対象の現在設定全体の指紋である。

公開処理が成功した場合は、既存の `ConversionBuildSnapshotService` / `BuildContext` が実行開始時に固定した設定を使い、実際のモードが包含する項目を反映済みにする。変換失敗を含むWARNINGでも、公開処理が成功していればFULL/DIFF/TEMPLATEの対象設定を反映済みにする。WARNINGのジョブ状態と変換失敗の履歴は維持する。FAILED、CANCELLED、HTML生成・公開失敗では反映状態を進めない。実行中に追加保存された値は未反映のまま残り、TEMPLATE/DIFFだけでは初期ビルド完了にならない。無変更DIFFが正常終了した場合も対象範囲の状態を更新するが、公開日時は従来どおり維持する。

保存と成功履歴更新ではライブラリ行をロックし、設定JSONの同時上書きを防ぐ。失敗・キャンセル・再起動時は最後の成功状態を基準に、一覧から再予約するかビルド管理から再実行できる。

## APIと画面

- 作成・更新・一覧・詳細の `LibraryDTO.buildRequirement` に、必要モード（不要ならnull）、初期ビルド要否、設定指紋、変更項目と日本語の理由を返す。
- 保存後の案内は `FormModal` を使い、元のフォームを閉じて一覧を保持する。予約の失敗は設定保存の失敗として扱わない。
- 案内からの `POST /api/builds/reserve` は `expectedConfigVersion` を付ける。既存の認可とライブラリロック下で指紋・必要モードを照合し、不一致なら409を返す。画面は最新の詳細を再取得して確認内容を更新する。
- 従来の手動予約APIの呼び出しには `expectedConfigVersion` を要求しない。予約後の進捗・エラー・再実行は既存のビルド管理を使用する。
- ジョブは従来どおり実行開始時に最新設定を固定する。待機中の編集でより強い処理が必要になった場合、そのビルドでは包含できない要求を残す。

## 既存データ・入出力

DBスキーマ変更は不要。反映状態のない既存ライブラリは、成功ビルド日時があれば最初の編集直前を移行基準とする。導入前から存在する未反映変更を遡って推定することはできない。成功日時がなければ初期フルビルドを案内する。

初期登録・コピー・インポートは新しいライブラリの状態として扱う。エクスポートでは反映状態を除外し、インポートでは反映状態とビルド成功履歴を移行元から引き継がない。設定更新APIでも、クライアント指定の反映状態を採用せずサーバーの状態を保持する。

テンプレート本体・ビューア登録・変換プロファイルの内容・ストレージ接続先など、ライブラリ以外の画面で行われた変更の案内は対象外。既存ビルドの差分検出や手動ビルドを利用する。

## 実装と検証

判定は `LibraryBuildRequirements`、保存時の基準保持は `LibraryServiceImpl`、公開成功時の確定と全HTML生成は `BuildPipelineOrchestrator`、案内の整合確認は `BuildServiceImpl` が担当する。画面は `LibraryBuildPrompt` を一覧と保存後で共有する。

`LibraryBuildRequirementsTest` は初回、見送り後の再読込、元の値への復帰、複数変更、部分モードの完了、実行中の変更、AI・運用設定、旧データ移行を検証する。関連するサービス、予約、公開パイプライン、入出力の既存テストも合わせて確認する。
