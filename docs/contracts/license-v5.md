# ライセンスファイル v5：登録時のホスト名

関連: [#6811](https://developer.astack.jp/redmine/issues/6811)、[v4契約](license-v4.md)、[製品ライセンス仕様](../specs/licensing.md)。

## 署名形式

envelopeは `version`（整数5）、`keyId`、`payload`、`signature` の4項目だけ。Ed25519の署名対象はUTF-8の `ASTACK-LICENSE-V5\n{keyId}\n{payload}`。payloadとsignatureはパディングなしの正規base64url。ファイル上限64KiB、厳密JSON・文字列・日時・利用枠・オプション・製品バージョン範囲の規則はv4を継承する。

payloadはv4の必須11項目に `hostName` を加えた12項目。OSDCを同梱する場合は `osdcLicense` を加えた13項目とする。`hostName` は必須でnull・空文字・未知項目を認めない。`productVersion` はv4と同じく必須。OSDCはv3と同じ規則で、省略が同梱なしを表し、nullは不可。

```json
"hostName": "astack-server.example"
```

署名領域とenvelopeのversionをv5として区別する。v4のpayloadに `hostName` だけを追加したファイルや、v4/v5の署名領域・versionを入れ替えたファイルは拒否する。ホスト名を含めたpayloadを署名し、ファイルの改変を検知する。

## ホスト名の正規化と用途

発行担当者は製品のライセンス管理画面からコピーしたインストールIDとホスト名を入力する。発行側は次の規則でホスト名を正規化してから署名する。

- Javaの空白判定に相当する前後空白を除去し、ASCII英字を小文字化する。末尾のドット1個を除去する。
- 正規化後はASCIIのDNS形式で最大253文字。各ラベルは1〜63文字の英小文字・数字・ハイフンで、先頭と末尾は英小文字または数字とする。空ラベル、空白、制御文字、国際化ドメイン名の非ASCII表記は認めない。
- `localhost`、`localhost.localdomain`、`localhost6`、`localhost6.localdomain6`、`localdomain`、`none`、`unknown`、`unknown-host`、`hostname`、`null`、`undefined` および `.localhost` で終わる名前は認めない。

製品の検証側は正規化後の正規表記との完全一致を要求する。署名payloadに大文字や前後空白、末尾ドット等が残った値を、黙って正規化して受理しない。OS取得側の規則は [MachineEnvironment.java](../../backend/src/main/java/jp/astack/backend/license/MachineEnvironment.java) を参照する。

`hostName` は署名された管理情報であり、追加の認証条件ではない。認証対象は引き続き署名済み `installationId` と、現在のOS情報・初回日時から導出したIDとの一致である。管理情報のホスト名と現在のOSホスト名を別に表示し、ホスト名の文字列だけで認証の可否を判定しない。

## 製品API・表示・互換性

管理者向け詳細状態と登録前プレビューのトップレベル `hostName` は、署名検証済みのv5情報を返す。共通の詳細表示で「登録時のホスト名」と表示する。現在のOS値は既存の `machineIdentity.hostName` に保持し、登録情報へ置き換えない。環境不一致で `INVALID` となった場合も、署名検証済みの登録ホスト名を表示して比較できる。

v1〜v4、自動体験版、未登録状態の `hostName` はnull。画面は「記録なし」と表示し、現在のOSホスト名を過去の登録情報として補完しない。一般ユーザー向けsummaryにホスト名を追加しない。日本語・英語・ベトナム語のラベルを共通辞書で翻訳し、登録されたホスト名自体は翻訳しない。

旧v1〜v4ファイルの署名形式と元ファイルを維持し、後から `hostName` を付加・再署名しない。旧ファイルの受理には既存の信頼鍵・製品版・期限・インストールID等の条件が引き続き必要。自動体験版の期間・権利・枠は変更しない。v5未対応の製品はv5を受理できないため、発行先製品を更新してから利用する。
