# ライセンスファイル v6：社内開発用共通ライセンス

関連: [#6830](https://developer.astack.jp/redmine/issues/6830)、[製品ライセンス仕様](../specs/licensing.md)、[v4契約](license-v4.md)。既存v1〜v5の形式・端末照合は変更しない。

## 署名と受理条件

envelopeは `version`（整数6）、`keyId`、`payload`、`signature` の4項目。Ed25519の署名対象はUTF-8の `ASTACK-LICENSE-V6\n{keyId}\n{payload}`。正規base64url、厳密JSON、上限64KiB、日時・機能・利用枠・製品バージョン範囲はv4の規則を継承する。

payloadはv4の必須11項目に `purpose` を追加する。`purpose` は文字列 `INTERNAL_DEVELOPMENT` のみ、`installationId` は明示的なnull、`expiresAt` は必須の有限日時とする。`hostName`・`osdcLicense`・未知項目を認めない。OSDC自体の端末条件や契約を緩和しない。ACPの権利は指定できるが、OSDCは別途その環境で管理する。

製品の通常の信頼鍵に加え、ビルド時の `ProductLicenseKeys.INTERNAL_DEVELOPMENT_KEY_IDS` に署名鍵IDが含まれる場合だけ受理する。通常ソースと通常ビルドの集合は空。Springプロファイル、環境変数、設定ファイル、アップロード内容でこの集合を追加しない。期限・開始時刻・製品版範囲・署名・利用枠の判定は通常どおり適用する。

v6では登録時・起動時・状態取得時に端末IDを照合しない。ホスト名・IPv4・初回日時の変更や端末情報取得不能は共通ライセンスを無効化しない。保存領域の書込権限や監査履歴の整合性は引き続き必要。旧体験版の開始日・初回日時を改変・再生成して延長しない。

v6の発行権利はオフラインのファイル所持に基づく。利用台数のオンライン制限や即時失効は提供しない。ファイルは社内向け配布先に限定し、顧客向け正式ビルドへ開発用受理鍵を組み込まない。

## API・画面

管理者向け詳細・登録前プレビューに `internalDevelopment`（検証済みの現在・候補ライセンスの用途）と `acceptsInternalDevelopment`（ビルドの受理可否）を追加する。一般ユーザー向けsummaryには用途や識別情報を追加しない。

共通ライセンスの `machineIdentity.bound`・`legacyLicense` はfalse、`registeredInstallationId`・`hostName` はnull。現在のOS情報は参考情報として既存項目で返す。画面の区分は「社内開発用（共通ライセンス）」とし、用途・期限を表示する。対応ビルドではID未取得でも登録可能だが、通常の端末ライセンスはサーバーで引き続き照合する。

## 発行・導入

発行サイトの用途で「社内開発用（共通ライセンス）」を選択し、対象製品版・契約者・エディション・権利・利用枠・有限期限を指定する。ホスト名とIDの入力は不要。発行サイトは用途を履歴・確認画面へ保存し、再ダウンロードは元ファイルを返す。

開発ビルドは通常の承認済み公開鍵・指紋の指定に加えて、受理を許可する鍵IDを明示する。

```bash
bash build/build-jar.sh --license-keys /path/to/trusted-keys.json \
  --license-key-id APPROVED_DEV_KEY --license-key-sha256 APPROVED_SPKI_SHA256 \
  --internal-development-key-id APPROVED_DEV_KEY
```

`--internal-development-key-id` は組込み対象鍵の部分集合で、複数指定できる。省略したビルドは同じ署名鍵を通常用途で信頼していてもv6を拒否する。正式配布の手順では指定しない。

発行ファイルをライセンス管理画面から登録する。新規構築時に自動配置する場合は、初回起動前に設定済み `astack.license.directory`（既定 `data/license`）へ `license.lic` として配置する。秘密鍵は配置しない。既存環境の更新は監査を残す登録API／画面を使い、稼働中にファイルを上書きしない。共通ファイルと初回日時をセットでコピーする必要はない。

更新時は新しい有限期限で発行した共通ファイルを各環境へ登録する。既存v1〜v5ファイルを用途変更・再署名して置き換えない。
