# 公開APIの接続元制限と認証ユーザー連携

ライブラリの「公開・配信 > リバースプロキシ連携」で、公開APIの接続元とアクセス履歴のユーザー識別を設定する。管理APIの認証・権限、公開コンテンツのACLは変更しない。

## 設定

| PathConfigの項目 | 既定値 | 内容 |
| --- | --- | --- |
| apiProxyEnabled | 新規OFF | 内部/外部の配信先と独立して中継連携を有効にする。明示値なしの旧設定では外部公開の接続元制限だけを継承する |
| apiAccessRestricted | false | 旧設定の読み込みと保存互換用。明示したapiProxyEnabledを優先する |
| apiAllowedPeers | 空文字 | IPv4・IPv6・CIDRを改行またはカンマ区切りで指定。8192文字・256件まで |
| apiIdentityHeaderEnabled | false | 許可した経路のユーザー名ヘッダーをアクセス履歴へ記録する |
| apiUserHeader | X-Astack-User | 最大128文字の専用HTTPヘッダー名 |

ホスト名・ポート付きIP・IPv6のゾーンIDは受け付けず、DNS検索は行わない。IPv4 mapped IPv6はIPv4へ正規化し、その形式のCIDRは/96〜128だけを受け付ける。中継連携ONでは信頼するIP/CIDRを必須とし、不正な値は保存・設定インポート時に拒否する。連携OFFでは保持値を検証・適用しない。ユーザー名記録ONでのみヘッダー名を検証し、OFF時は保持値を使用しない。Authorization・Cookie・Host・Forwarded・X-Forwarded-*・Content-*などの予約ヘッダーはユーザー名に使わない。

制限対象は `/pub/{libraryKey}/_api/**` の全メソッド（OPTIONSを含む）。未許可元・不正な有効設定・物理接続元を確認できない場合は403とする。設定変更は次の要求から反映する。静的ページ・管理API・短縮リンクのリダイレクト入口 `/s/{code}` はこの制限の対象外。リンク作成・原本取得などの `_api` は対象になる。静的コンテンツ自体の認証は公開先Webサーバで設定する。

ライブラリは名前の一致を優先し、一致がない場合だけ旧互換の数値IDとして解決する。未公開の数値名や同名の複数ライブラリがある場合は404とし、別のIDへ読み替えない。接続元の検査とダウンロードAPIで同じ対象を使う。

## 物理接続相手と閲覧PCのIP

TomcatのRequest.getPeerAddr()を専用Valveでサーバー属性へ保存し、許可元と照合する。ヘッダーやNativeで復元されたgetRemoteAddr()は接続元制限に使わない。Valveが作った属性を取得できなければ、有効な制限を迂回させない。パスはMVCと同じセグメント単位で一度復号し、生のmatrixパラメーターを除いて判定する。符号化されたセミコロンをライブラリ名の一部として扱う場合とは区別する。

アクセス履歴のclientIpは、従来どおりServletが確定した閲覧PCのIPを使用する。 ARRの既定設定で復元結果が数値IPv4:port・[数値IPv6]:portになる場合は、妥当なportを確認してIPを正規化する。portなしの数値IPv6を先に判定し、曖昧な末尾をportと推測しない。ホスト名・複数値・不正なendpointは採用しない。レート制限も正規化したIP単位とし、不正な値は共通のUNKNOWN枠へ集約する。元のHTTP転送ヘッダーをこの処理で読むことはない。プロキシ配下ではTomcatのNative方式と明示した信頼プロキシ設定で復元する。ライブラリの許可元指定だけでは、グローバルの信頼プロキシを自動追加しない。

例としてIISからTomcatへ直接中継する場合、ライブラリの許可元にはIISのIPを指定し、Tomcatのinternal-proxiesにも運用構成に合わせてIISを追加する。ほかのプロキシを挟む場合は、Astackの実接続相手が変わるので信頼境界を再設計する。Viteのループバックを許可するだけで、その先のIISを認証済みと判断してはならない。

参照: [Tomcatの接続相手と復元後IPの区別](https://tomcat.apache.org/tomcat-10.1-doc/config/valve.html#Remote_CIDR_Valve)、[Spring Bootのプロキシ設定](https://docs.spring.io/spring-boot/3.5/how-to/webserver.html#howto.webserver.use-behind-a-proxy-server)。

## ユーザーの識別

制限を通過した要求で認証連携がONの場合だけ、設定したヘッダーを採用する。単一の非空値・255文字以内とし、制御文字・改行区切り・複数値は採用しない。識別情報はサーバー側request属性から取得し、対象ライブラリとの一致を確認する。

確認できたユーザーはidentityStatus=AUTHENTICATED、identitySource=TRUSTED_PROXY、accountNameに記録する。ブラウザ通知とサーバーが観測した原本転送の両方で使用する。ヘッダーなし・不正値・連携OFFの場合はUNKNOWNのまま記録し、匿名とは断定しない。同じイベントの再送で既存の識別情報を上書きしない。既存のDB列とOpenSearch mappingを使用し、データ移行は不要。

この情報はアクセス履歴の識別専用で、Astackのログイン・管理権限・文書ACLのAuthenticationには使わない。信頼するプロキシは管理下のIPに限定し、外部からの同名ヘッダーを除去して、実際に認証したユーザーで上書きする。

## 生成コンテンツと認証

SDK版3の配布設定sameOriginAuthenticationは、認証連携ONのライブラリでだけ有効になる。公開ページとAPIのオリジンが一致する場合だけcredentials: same-originを使用する。連携OFF・別オリジンの直接送信は従来どおりomitで、資格情報を任意の別ホストへ送らない。IIS等の認証を利用する際は、公開ページと同じオリジンにAPI中継パスを設ける。

認証連携・SDK版の変更はDIFFの生成判定へ含める。APIパスや認証連携モードの変更後は再ビルド・再公開が必要。署名済み対象Contextの形式は変えず、旧SDK・署名との互換性を維持する。

[IIS向け設定キット](../../scripts/iis-auth/README.md)を参照。S3など認証処理のない静的配信だけではユーザーを識別できないため、認証を持つゲートウェイまたは別の認証連携が必要。
