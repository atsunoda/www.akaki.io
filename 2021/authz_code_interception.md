---
description: 'Jul 5, 2021'
---

# OAuthにおける認可コード横取り攻撃とその対策

[前回の記事](url_scheme_hijack.md)で示したように、カスタムURLスキームを偽装した不正アプリは正規アプリへのディープリンクを乗っ取れる。この挙動の悪用シナリオとして、正規アプリと認可サーバー間のOAuthフローにおける認可コード横取り攻撃が知られている。この攻撃への対策を把握するためにiOS環境でシナリオを再現し、PKCEの有効性を確認した。

## 要約

OAuth 2.0の拡張機能であるPKCEを導入することで認可コード横取り攻撃を無効化できる。OAuth 2.0の仕様では、認可サーバーはネイティブアプリをクライアント認証できない。そのため、認可サーバーは認可コードを横取りした不正アプリと正規アプリを識別できない。しかし、PKCEの仕組みにより認可サーバーは正規アプリを識別できるようになり、認可コード横取り攻撃の検知が可能となる。

## ネイティブアプリにおけるクライアント認証の課題

ネイティブアプリのOAuthフローには認可コード付与方式が推奨されている。OAuthはアクセス権の移譲プロトコルであり、RFC 6749で定義されたOAuth 2.0が業界標準となっている¹。OAuth 2.0の仕様では、アクセストークンを発行する際の認可付与の方式として、一時的なコードにより移譲への同意を伝える「認可コード付与方式」を含む4つの方式を定義している。さらに、クライアントタイプとして「機密」と「公開」の2つを定義し、ネイティブアプリを公開クライアントに分類している。このように定義された認可付与の方式とクライアントタイプの特性を考慮し、OAuth 2.0の専門書である『OAuth 2 in Action』（Manning）ではネイティブアプリのOAuthフローに認可コード付与方式を推奨している²（図1）。

![&#x56F3;1. &#x8A8D;&#x53EF;&#x30B3;&#x30FC;&#x30C9;&#x4ED8;&#x4E0E;&#x65B9;&#x5F0F;&#x306B;&#x3088;&#x308B;&#x30CD;&#x30A4;&#x30C6;&#x30A3;&#x30D6;&#x30A2;&#x30D7;&#x30EA;&#x306E;OAuth&#x30D5;&#x30ED;&#x30FC;](../.gitbook/assets/22_figure1.png)

ただし、ネイティブアプリのOAuthフローでは認可サーバーがクライアント認証できない。OAuth 2.0の仕様では、パスワードなどのクレデンシャルによるクライアント認証の機会を認可サーバーへ提供している。しかし、認可サーバーによるネイティブアプリの識別を目的とするクライアント認証を禁止している。なぜなら、ネイティブアプリに組み込まれたクレデンシャルは、リバースエンジニアリングや通信解析により機密性を維持できないからである。

クライアント認証できない場合、認可サーバーはリダイレクトURIに基づいてネイティブアプリを識別する。OAuth 2.0の仕様におけるセキュリティ上の検討事項では、クライアント認証が不可能な場合、認可サーバーはクライアントを識別するための他の手段を採用するよう求めている。具体的な手段として、認可サーバーはクライアントへのリダイレクトURIを事前に登録しておき、認可要求に含まれるリダイレクトURIと比較することを挙げている。これにより、認可サーバーはリダイレクトURIが一致しないアプリを不正と判断し、認可コードの付与を拒否できるようになる。

## カスタムURLスキームによるアプリ識別のリスク

リダイレクトURIにカスタムURLスキームが使用されると、認可サーバーはネイティブアプリを識別できなくなる。OAuth 2.0の仕様では、ネイティブアプリがリダイレクトURIにカスタムURLスキームを使用することを認めている。しかし、AndroidとiOSはカスタムURLスキームに基づいてアプリを識別できない³。そのため、認可サーバーが正規アプリへのリダイレクトURIの一致を確認していても、OSの処理によって認可コード付きのURLが不正アプリへリダイレクトされる。その結果、認可コードを横取りした不正アプリは、正規アプリになりすましてアクセストークンを発行できる。このような不正アプリによる行為を「認可コード横取り攻撃」という（図2）。

![&#x56F3;2. &#x8A8D;&#x53EF;&#x30B3;&#x30FC;&#x30C9;&#x6A2A;&#x53D6;&#x308A;&#x653B;&#x6483;&#x306E;&#x30D5;&#x30ED;&#x30FC;](../.gitbook/assets/22_figure2.png)

### iOS環境での認可コード横取り攻撃

カスタムURLスキームを偽装したiOSアプリを使用して、LINEログインにおける認可コード横取り攻撃を実証した。LINEログインは、OAuth 2.0とOpenID Connectに基づいたLINEアカウントによるソーシャルログイン機能である⁴。検証では、LINEログインを搭載した架空の決済アプリ「Akapay」とその偽アプリを使用して、偽アプリが横取りした認可コードを用いてアクセストークンを発行し、被害者のLINEアカウントのプロフィール情報を窃取するシナリオを再現した。事前に両方のアプリで同じカスタムURLスキーム `line3rdp.io.akaki.akapay://` を宣言し、このスキームをLINEログインのチャネルに登録している。Akapayアプリ、偽アプリの順にXcode経由でiOS 14.6の端末にインストールして検証を実施した。その結果、AkapayアプリからSafari経由でLINEログイン後、LINEアカウントのプロフィール情報へのアクセスをAkapayへ認可すると、認可コード付きのURLが偽アプリに横取りされ、アクセストークンの発行とプロフィール情報へのアクセスを許した（図3）。

![&#x56F3;3. iOS&#x30A2;&#x30D7;&#x30EA;&#x306B;&#x3088;&#x308B;&#x8A8D;&#x53EF;&#x30B3;&#x30FC;&#x30C9;&#x6A2A;&#x53D6;&#x308A;&#x653B;&#x6483;](../.gitbook/assets/22_figure3.gif)

認可コードを横取りした偽アプリは、Akapayアプリになりすましてアクセストークンを要求する。SafariからLINEログインした後、Akapayへの認可画面を表示する際のGETリクエストが認可要求である。認可要求の `redirect_uri` にはAkapayアプリのカスタムURLスキームが含まれる。このスキームはLINEログインのチャネルに登録しているため、認可サーバーは正規アプリへのリダイレクトURIとして判断する。

```text
GET /oauth2/v2.1/authorize/consent
    ?response_type=code
    &client_id=██████████
    &scope=profile
    &state=WyzSEFbiQojxI0nKJ5gyuJVStgxeTify57e25kHPR7o
    &redirect_uri=line3rdp.io.akaki.akapay://authorize/ HTTP/1.1
Host: access.line.me
...
```

Akapayを認可するとPOSTリクエストが発生し、それに対するレスポンスのLocationヘッダに認可コードである `code` が付与される。

```text
POST /oauth2/v2.1/authorize/consent HTTP/1.1
Host: access.line.me
...
```

```text
HTTP/1.1 302 Found
location: line3rdp.io.akaki.akapay://authorize/
    ?code=7hrxXGkHnAlmgRg3oN6z
    &state=WyzSEFbiQojxI0nKJ5gyuJVStgxeTify57e25kHPR7o
...
```

Locationヘッダに設定されるカスタムURLスキームは偽アプリに乗っ取られているため、リダイレクトにより認可コードが偽アプリに横取りされる。偽アプリは横取りした認可コードを含めたアクセストークン要求を認可サーバーへ送信する。認可サーバーは偽アプリからの不正な要求を検知できずにアクセストークンを発行する。

```text
POST /oauth2/v2.1/token HTTP/1.1
Host: api.line.me
...

client_id=██████████
&grant_type=authorization_code
&code=7hrxXGkHnAlmgRg3oN6z
&redirect_uri=line3rdp.io.akaki.akapay://authorize/
```

```text
HTTP/1.1 200 OK
...

{
    "access_token": "eyJhbGciOiJIUzI1NiJ9._utnquoDbhxVaFkT-PF479x-oG1vsz04QDuMQfO-HTiDY4cbBu8rR943iNT6TC5f3-cijsdUzSPCHjllhnC_x9At2LMJ1lYsZjNoRfbfzCERlgVnP_P7RZTiGYL85h0mN3zlaAyCTJi1xwO5av7twf1iA9PGspz1PWpIHEHetHA.QkR9WDRrR_r4tjIp3SJF-b4OAmJ1r83uMLXVzRDpDB4",
    "token_type": "Bearer",
    "refresh_token": "XewGVk2mNAE2gQHKwsIt",
    "expires_in": 43200,
    "scope": "profile"
}
```

発行されたアクセストークンを使用して、偽アプリは被害者のLINEアカウントのプロフィール情報にアクセスできる。

```text
GET /v2/profile HTTP/1.1
Host: api.line.me
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9._utnquoDbhxVaFkT-PF479x-oG1vsz04QDuMQfO-HTiDY4cbBu8rR943iNT6TC5f3-cijsdUzSPCHjllhnC_x9At2LMJ1lYsZjNoRfbfzCERlgVnP_P7RZTiGYL85h0mN3zlaAyCTJi1xwO5av7twf1iA9PGspz1PWpIHEHetHA.QkR9WDRrR_r4tjIp3SJF-b4OAmJ1r83uMLXVzRDpDB4
...
```

```text
HTTP/1.1 200 OK
...

{
    "userId": "█████████████████████████████████",
    "displayName": "Alice"
}
```

## PKCEによる認可コード横取り攻撃の検知

PKCEの導入により認可サーバーは認可コードを横取りした不正アプリを検知できるようになる。PKCE（Proof Key for Code Exchange）は、認可コード横取り攻撃の緩和策としてRFC 7636で定義されたOAuth 2.0の拡張機能である⁵。PKCEを導入したOAuthフローでは、認可要求とアクセストークン要求を送信したネイティブアプリが同じかどうかを認可サーバーが判断できる。その仕組みは、まず正規アプリは認可要求の際に一時的な鍵を生成し、その鍵を変換したチャレンジを認可サーバーに送信する。その後、アクセストークン要求の際に鍵を認可サーバーに送信する。認可サーバーは受信した鍵からチャレンジを計算し、認可要求の際に受信したチャレンジと比較する。チャレンジが一致すれば同じアプリからの要求であることが証明されるため、認可コードを付与した正規アプリにのみアクセストークンを発行できるようになる。したがって、不正アプリは認可コードを横取りできても鍵を送信できないため、認可サーバーにアクセストークンの発行を拒否される（図4）。

![&#x56F3;4. PKCE&#x306B;&#x3088;&#x308B;&#x8A8D;&#x53EF;&#x30B3;&#x30FC;&#x30C9;&#x6A2A;&#x53D6;&#x308A;&#x653B;&#x6483;&#x306E;&#x691C;&#x77E5;&#x30D5;&#x30ED;&#x30FC;](../.gitbook/assets/22_figure4.png)

### iOS環境でのPKCEによる攻撃の無効化

AkapayアプリにPKCEを導入することで、LINEログインにおける認可コード横取り攻撃を無効化した。LINEログインの認可サーバーはPKCEに対応しており⁶、iOS Swift版のLINE SDKのログイン処理は標準でPKCEを実装している⁷。この実装を参考にAkapayアプリへPKCEを導入し、同様の悪用シナリオの再現を試みた。その結果、認可コード付きのURLは偽アプリに横取りされるものの、アクセストークンの発行とプロフィール情報へのアクセスは防止できた（図5）。

![&#x56F3;5. PKCE&#x306B;&#x3088;&#x308B;&#x8A8D;&#x53EF;&#x30B3;&#x30FC;&#x30C9;&#x6A2A;&#x53D6;&#x308A;&#x653B;&#x6483;&#x306E;&#x7121;&#x52B9;&#x5316;](../.gitbook/assets/22_figure5.gif)

PKCEの仕組みにより、Akapayアプリになりすました偽アプリからのアクセストークン要求は拒否される。PKCEを導入したAkapayアプリからの認可要求には、Akapayアプリが生成した鍵を変換したチャレンジである `code_challenge` と、その変換方式を示す `code_challenge_method` が含まれる。

```text
GET /oauth2/v2.1/authorize/consent
    ?response_type=code
    &client_id=██████████
    &scope=profile
    &code_challenge=0AqK7nui3rIfvnvEzHArPoQ2YEPoWUM8Y65b_t182Eg
    &code_challenge_method=S256
    &state=kxlOyCxyfUX74G6L-46gP7EI0ClpWRmBTaknqYZVBxo
    &redirect_uri=line3rdp.io.akaki.akapay://authorize/ HTTP/1.1
Host: access.line.me
...
```

Akapayを認可した際のLocationヘッダに設定されるURLの構成はPKCEを導入する前と変わらない。そのため、PKCEを導入してもカスタムURLスキームを乗っ取られ、偽アプリに認可コードを横取りされる。

```text
POST /oauth2/v2.1/authorize/consent HTTP/1.1
Host: access.line.me
...
```

```text
HTTP/1.1 302 Found
location: line3rdp.io.akaki.akapay://authorize/
    ?code=HJyxHiWX1NJeYjyLRiVG
    &state=kxlOyCxyfUX74G6L-46gP7EI0ClpWRmBTaknqYZVBxo
...
```

しかし、偽アプリはAkapayアプリが生成した鍵を知らないため、アクセストークン要求の際に正しい鍵を `code_verifier` として送信できない。認可サーバーは正しい鍵からチャレンジを計算できないことから、不正な要求と判断してアクセストークンの発行を拒否する。アクセストークンが発行されないため、偽アプリは被害者のLINEアカウントのプロフィール情報にアクセスできない。

```text
POST /oauth2/v2.1/token HTTP/1.1
Host: api.line.me
...

client_id=██████████
&grant_type=authorization_code
&code=HJyxHiWX1NJeYjyLRiVG
&code_verifier=
&redirect_uri=line3rdp.io.akaki.akapay://authorize/
```

```text
HTTP/1.1 400 Bad Request
...

{
    "error": "invalid_grant",
    "error_description": "code_verifier does not match"
}
```

## 所感

ネイティブアプリにおけるOAuth 2.0のベストプラクティスを定義したRFC 8252や、OAuth 2.0におけるセキュリティのベストプラクティスを定義するドラフトでは、ネイティブアプリのOAuthフローへのPKCEの導入を必須としている⁸ ⁹。さらに、ドラフトでの改善が進むOAuth 2.1では認可コード付与方式の標準機能としてPKCEを導入している¹⁰。PKCEの普及により認可コード横取り攻撃の問題は解決する。しかし、カスタムURLスキームを乗っ取った不正アプリが偽のログインを要求するなどの攻撃への懸念は残る。RFC 8252や現状のOAuth 2.1では、リダイレクトURIにはカスタムURLスキームよりも、ユニバーサルリンクのような乗っ取りが困難なHTTP URLスキームの使用を推奨している。また、OAuth 2.0とOpenID Connectに基づいた主に金融API向けのセキュリティ要求事項であるFAPI（Financial-grade API）では、リダイレクトURIへのカスタムURLスキームの使用を認めていない¹¹。現時点では、金融レベルのセキュリティが求められるアプリのOAuthフローからカスタムURLスキームの廃止が進んでいた。

¹ [RFC 6749 - The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/rfc6749/)  
² [6.1.5. Choosing the appropriate grant type - OAuth 2 in Action](https://livebook.manning.com/book/oauth-2-in-action/chapter-6/113)  
³ [カスタムURLスキームの乗っ取りとその対策 - Akaki I/O](url_scheme_hijack.md)  
⁴ [LINEログインの概要 - LINE Developers](https://developers.line.biz/ja/docs/line-login/overview/)  
⁵ [RFC 7636 - Proof Key for Code Exchange by OAuth Public Clients](https://datatracker.ietf.org/doc/rfc7636/)  
⁶ [LINEログインをPKCE対応する - LINE Developers](https://developers.line.biz/ja/docs/line-login/integrate-pkce/)  
⁷ [Feature PKCE · line/line-sdk-ios-swift - GitHub](https://github.com/line/line-sdk-ios-swift/pull/133)  
⁸ [RFC 8252 - OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/rfc8252/)  
⁹ [OAuth 2.0 Security Best Current Practice - draft-ietf-oauth-security-topics-18](https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/18/)  
¹⁰ [The OAuth 2.1 Authorization Framework - draft-ietf-oauth-v2-1-02](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/02/)  
¹¹ [Final: Financial-grade API Security Profile 1.0 - Part 1: Baseline](https://openid.net/specs/openid-financial-api-part-1-1_0-final.html)

