# ライセンスファイル v4：製品バージョンと署名鍵

関連: [#6747](https://developer.astack.jp/redmine/issues/6747)、[v2契約](license-v2.md)、[OSDC同梱のv3契約](license-v3.md)、[製品ライセンス仕様](../specs/licensing.md)。

新規の登録時ホスト名付きファイルは [v5契約](license-v5.md) を使う。本書は既存v4の形式を定義し、v4へ `hostName` を後付けしない。

## 署名形式

envelopeは `version`（整数4）、`keyId`、`payload`、`signature` の4項目だけ。Ed25519の署名対象はUTF-8の `ASTACK-LICENSE-V4\n{keyId}\n{payload}`。payloadとsignatureはパディングなしの正規base64url。ファイル上限64KiB、厳密JSON・文字列・日時・利用枠・オプションの規則はv2を継承する。

payloadはv2の10項目に、必須の `productVersion` を加えた11項目。OSDCを同梱する場合はさらに `osdcLicense` を加えた12項目とする。OSDCの項目・ハッシュ・サイズ・OS/OSDC版との照合・配置・秘匿はv3と同じ。`osdcLicense: null` は不可、省略で同梱なしを表す。

```json
"productVersion": {"min": "4.0.0", "maxExclusive": "4.1.0"}
```

`productVersion` はこの2項目だけを持つ。各版はASCII数字の `major.minor.patch` の3要素、それぞれ0〜999999。0以外の先頭0、符号、接頭辞、空白、プレリリース、ビルドサフィックスを認めない。各要素を整数として比較し、`min < maxExclusive` を要求する。署名検証後に `min <= 実行製品版 < maxExclusive` を確認し、範囲外・実行版不明なら無効として拒否する。OSDCの2要素 `osdcLicense.productVersion` とは別の値である。

## 実行製品版と鍵の配布

製品検証にはソースの `ProductBuildVersion.VERSION`（現行 `4.0.0`）を使う。`astack.version`、表示バージョン、環境変数、アップロードファイルの値は実行版に採用しない。Maven artifactの `0.0.1-SNAPSHOT` も製品版ではない。製品リリース時にはこの定数を実際の版へ更新し、対応公開鍵と同じレビュー・ビルドで配布する。設定変更だけで対応版を偽装できない。実行ファイル自体の改変を防ぐ仕組みではない。

信頼鍵はJavaコードの `ProductLicenseKeys.PUBLIC_KEYS` へ直接組み込む。形式は `Map<String, String>` の `keyId → Ed25519 SPKI公開鍵PEM`、keyIdは `[A-Za-z0-9_-]{1,100}`。起動時にこの定数から公開鍵を生成する。外部ファイル・classpathリソース・環境変数・システムプロパティから鍵を追加・上書きしない。旧classpathリソース `license/trusted-keys.json` は廃止し、同名リソースを配置しても読み込まない。

リリースJARは `build/build-jar.sh` に発行システムから取得した公開鍵JSON、組み込むkeyId、別途承認したSPKI DERのSHA-256指紋を渡して作成する。例えば1鍵なら `./build/build-jar.sh --license-keys trusted-keys.json --license-key-id KEY_ID --license-key-sha256 APPROVED_SHA256` とする。JSONは `keyId → Ed25519 SPKI公開鍵PEM` のオブジェクトで、複数鍵を使う場合はkeyIdと指紋を鍵ごとに繰り返し指定する。スクリプトは指定鍵の形式・指紋を検証し、作業ツリーを変更せず一時的なバックエンドソースへ組み込んでビルドする。生成JARに指定した鍵が含まれることを確認するまで `Astack.jar` と `backend/target/` の配布JARを更新しない。引数の省略、keyIdの欠落、JSON重複キー、不正形式、指紋不一致ではビルドを中断する。JSON自体はJARに同梱せず、起動時の鍵追加にも使用しない。

指紋はJSONと同じ取得元から無検証で転記せず、発行システム管理者等が別経路で承認した値を指定する。鍵の承認・廃止と `ProductBuildVersion.VERSION` を同じリリースで確認する。秘密鍵・テスト鍵を製品へ配置しない。通常ソースの `PUBLIC_KEYS` は空のままとし、信頼鍵はリリースビルド時に設定する。

この方式は公開鍵リソースの編集だけでの差し替えを防ぐための簡易対策であり、クラスの解析・改変や検証処理の迂回を完全に防止するものではない。二段署名・ネイティブ化・難読化は導入しない。

新しい製品版で旧鍵を廃止する場合、その版のコード定数から旧keyIdを除去する。既存キーを互換性のために自動追加しない。旧keyIdのファイルは署名が正しくても検証できなくなる。コード定数に複数鍵を含める場合、含めた各鍵が署名した範囲内のv4ファイルは受理可能になるため、配布対象を明示してレビューする。鍵名から対応版を推測せず、署名済み範囲も常に検証する。

## インストールIDの方式

[#6809](https://developer.astack.jp/redmine/issues/6809) 以降の新規登録では、製品管理画面に表示する現在のUUIDv8を `installationId` へ指定する。現在のOS認識ホスト名・認証対象IPv4・初回日時をSHA-256で導出したIDであり、署名payloadの項目やenvelopeのversionを変更しない。UUIDv8の「8」と本契約の署名形式「v4」は別の値である。

署名検証済みの旧UUIDv4登録ファイルは旧ID一致時の経過措置として継続認証するが、新規アップロードでは現在の新IDが必須。再発行は管理画面のIDとホスト名で依頼する。DHCPを含むIPv4変更・ホスト名変更・初回日時が異なる再インストールでは再発行が必要。保存データと同じ端末情報での復元、旧形式移行の日時採用、自動体験期間を延長しない条件は [製品ライセンス仕様](../specs/licensing.md#端末に紐付くインストールid) を参照する。

## 互換性・状態

v1〜v3は従来形式を維持し、対応範囲を後付けしない。これらの受理には該当するkeyIdが組込み定数に必要。v4固有の `productVersion` を旧形式へ追加したものは拒否する。

検証済みのプレビュー・管理者向け状態APIは `productVersion: {min,maxExclusive}` を返す。旧形式・自動体験版・無効ファイルはnull。範囲外のファイルは `INVALID` となり新規更新を許可せず、正式ライセンス利用後に自動体験版へ戻さない。保存済みの正式ファイルを削除・再署名しない。既存公開文書の閲覧方針は従来どおり。

既存の2引数 `LicenseVerifier(keys, clock)` は旧形式向けの互換APIであり、実行版が不明なためv4を拒否する。製品のbeanは必ず組込み版を指定する。
