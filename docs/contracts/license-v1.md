# ライセンスファイル v1 検証基盤

関連: [#6713](https://developer.astack.jp/redmine/issues/6713)、[再設計PR #49](https://github.com/nt-bsys-dev/Astack4/pull/49)。本書は検証基盤の実装仕様。業務API・ワーカー・管理画面への統合と配備前提は [製品ライセンス仕様](../specs/licensing.md) を参照する。

新規発行は [v2契約](license-v2.md) を使う。本書は旧v1の保存形式を定義する。現製品では旧 `maxUsers` を検証するがユーザー数枠として適用せず、プロジェクト枠は無制限とする。旧ユーザー枠をプロジェクト枠へ読み替えない。

## ファイル形式

ファイル全体は64KiB以下のUTF-8 JSON。envelope の必須項目は次の4つだけとし、未知項目、重複キー、末尾の追加JSONを拒否する。

| 項目 | 値 |
| --- | --- |
| `version` | 整数1 |
| `keyId` | 1〜100文字、ASCII英数字・`_`・`-`。呼出側の信頼鍵一覧から選択 |
| `payload` | 権利JSONのUTF-8バイト列をbase64url、パディングなしで表現 |
| `signature` | Ed25519署名64バイトのbase64url、パディングなし |

署名対象のバイト列は UTF-8 の `ASTACK-LICENSE-V1\n{keyId}\n{payload}`。改行はLF。payload はファイル内の文字列をそのまま使用し、JSONを再整形して署名・検証しない。未対応のversionは拒否する。ファイル内でアルゴリズムや公開鍵を指定できない。

payload の必須項目は以下の10項目。未知項目・重複キー・末尾の追加JSONを拒否する。

| 項目 | 入力条件 |
| --- | --- |
| `licenseId` | 1〜100文字、空白のみ不可 |
| `product` | `ASTACK4` |
| `edition` | 1〜32文字、`[A-Z][A-Z0-9_]*`。本段階では表示用であり、販売権利の対応付けはしない |
| `issuedTo` | 1〜300文字、空白のみ不可 |
| `issuedAt` | UTCの秒精度 `YYYY-MM-DDTHH:mm:ssZ` |
| `notBefore` | 同上。開始時刻を含む |
| `expiresAt` | 同上、またはnull。終了時刻を含まない。issuedAt/notBeforeより後 |
| `installationId` | 小文字の標準UUID表記、対象インストールと一致。ワイルドカード不可 |
| `features` | 下記の機能の配列。空配列可、重複・未知機能不可 |
| `limits` | `maxLibraries`・`maxDocuments`・`maxUsers`のみ。値は0〜9223372036854775807の整数、またはnull |

機能は `BUILD_PIPELINE`、`OPENSEARCH`、`AI_SEARCH`、`CONVERTER_PLUGIN`、`DOCUMENT_PORTAL_KIT`、`ACCESS_LOG_AGGREGATION`。nullの上限は無制限、0は利用枠なし。発行側でJavaScriptの安全な整数範囲を超える値を扱う場合は、数値の丸めが起きないJSON生成・入力検証が必要。一般的な販売上限・edition別の権利は別途固定する。

文字列の長さはJavaのUTF-16コード単位で計数する。制御文字、不正UTF-8、不正なサロゲートを拒否する。入力やパース例外の内容をエラーへ含めない。署名不正・未知鍵・形式不正は `LicenseValidationException` へまとめ、時刻による状態は別途返す。

## 実装と統合契約

- `LicenseVerifier` に公開鍵の不変Mapと `Clock` を渡す。製品用の信頼鍵登録は後続段階で行い、旧ブランチの露出鍵を追加しない。任意の公開鍵をアップロードから登録しない。
- 検証済み `LicenseClaims` は不変。`state()` は毎回Clockを参照し、issuedAt/notBeforeの前は `NOT_YET_VALID`、終了時刻以後は `EXPIRED`、その他は `ACTIVE`。
- `InstallationIdentity.loadOrCreate()` はデータ領域に `installation.id` を保存する。プロセス間ファイルロックと同一JVM内の排他で初回生成を直列化し、再起動時は同じIDを読む。
- IDの形式不正、リンク、既存 `license.lic` がある状態でのID消失は復旧エラー。黙って再生成しない。IDを含む環境全体の複製を検出する機構ではない。
- `LicenseStore` は同じデータ領域を使う製品内の単一インスタンスとして構成する。複数プロセスが同時にライセンスを更新する運用は対象外。別プロセスの書換えを自動監視しない。
- `validate()` は検証のみ。`activate()` は同じファイルを再検証し、ACTIVEの場合だけ同一ファイルシステムの一時ファイルから原子的に置換する。書込みをforceしてから置換するが、突然の電源断に対する親ディレクトリエントリの永続化保証はOS依存。原子的移動非対応の場合は失敗させる。
- 不正更新や保存失敗は既存のメモリ状態を変えない。再起動時は期限切れ情報を含め署名を検証して読み直す。壊れたファイルは `INVALID` とし、未登録と区別する。
- 保存先は管理者が所有・保護するデータ領域とする。最終パスのシンボリックリンクを拒否するが、管理者が親ディレクトリを悪意を持って並行置換する攻撃までは防止しない。POSIXでは一時ファイル/置換先を所有者読書きのみにする。Windowsの領域ACLはインストーラー/運用で設定する。
- ファイル登録とDB監査、利用枠台帳、業務ポリシー、画面は後続段階。`LicenseStore` をControllerに直接接続せず、監査・権限・制限適用を持つ統合サービスを経由する。

## 公開テストベクトル

[テスト資材](../../backend/src/test/resources/license-v1/)の `node-generated.lic` はNode.js標準cryptoで生成したv1ファイル、`test-public-key.pem` はその検証専用公開鍵。秘密鍵は保存しない。製品の信頼鍵に登録しない。固定のインストールIDと注入ClockでJavaから検証する。

この資材は言語間の署名バイト列の契約確認に使う。単体テストの異常系はテストごとに別の鍵をメモリ内生成し、公開テストベクトル用の秘密鍵に依存しない。
