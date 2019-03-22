# Request-URIのパスからのオープンリダイレクト

以前、ロシアのバグハンターSergey Bobrov氏（[@Black2Fan](https://twitter.com/Black2Fan)）がHackerOneで公開しているレポートをもとに、[Request-URIのパスからのCRLFインジェクション](/2018/crlfi_via_path_of_request-uri.md)をまとめた。Bobrov氏はパスからの入力によるオープンリダイレクトのレポートも多数公開しており、脆弱性を調査する上で参考になった。そこで今回もまたBobrov氏のレポートをもとに、パスからのオープンリダイレクトの事例をまとめて紹介する。また最後にPythonを用いてパスからのオープンリダイレクトを簡単に体験する方法も紹介する。

## HackerOneで公開されている事例

Bobrov氏が公開している事例の多くは、30xリダイレクトの際にパスの値がLocationヘッダに出力される箇所で見つかっている。Locationヘッダの値は相対パスで出力される場合と絶対パスで出力される場合があり、外部サイトへの転送方法は場合により異なる。今回は相対パスで出力される場合に見つかったパスからのオープンリダイレクトの事例を紹介する<sup id="f1">[1](#fn1)</sup>。

相対パスで出力される場合でのリダイレクト処理の要因は、トレイリングスラッシュ（パスの末尾にある `/` ）に関するものが大半だったが、それ以外の事例もあった。リダイレクト処理ごとに事例を見ていこう。

### トレイリングスラッシュを削除するリダイレクトでの事例

#### [#160047](https://hackerone.com/reports/160047)

任意のパスからトレイリングスラッシュを取り除いた値が、相対パスとしてLocationヘッダに出力される箇所で脆弱性は見つかった。`GET /xxx/ HTTP/1.1` でリクエストすると `Location: /xxx` がレスポンスされたため、パスの先頭を `//` にすることでスキームを省略した絶対パスとしてブラウザに認識させ、外部サイトにリダイレクトさせている。

hxxps://apps.example.com//evil.com/

```http
HTTP/1.1 301 Moved Permanently
...
Location: //evil.com
```

### トレイリングスラッシュを追加するリダイレクトでの事例

#### [#192373](https://hackerone.com/reports/192373)

任意のパスにトレイリングスラッシュを追加するリダイレクトでも脆弱性は見つかっている。この事例では入力値制限によりパスの先頭を `//` にできなかったため、間に `HT`（%09）を含め `/ /` にすることで制限を回避したと考える。

hxxps://cooking.lady.example.jp/%09/evil.com

```http
HTTP/1.1 301 Moved Permanently
...
Location: / /evil.com/
```

また `//` の後方を `\`（%5C）に代え `/\` にすることでも制限を回避している。先頭が `/ /` や `/\` の場合も絶対パスとして認識されるのは一部のブラウザ特有の動作である<sup id="f2">[2](#fn2)</sup>。

hxxps://cooking.lady.example.jp/%5Cevil.com

```http
HTTP/1.1 301 Moved Permanently
...
Location: /\evil.com/
```

#### [#125000](https://hackerone.com/reports/125000)

任意のパスではトレイリングスラッシュを追加するリダイレクトが発生しなくても、パスの末尾を `/%2E%2E` や `/%2F..` にするとリダイレクトが発生する場合がある。`GET /xxx HTTP/1.1` では404だが、`GET /xxx/%2F.. HTTP/1.1` でリクエストすると `Location: /xxx/%2F../` がレスポンスされるような動作である。この動作を応用し、先頭が `//` になり末尾が `/%2F..` のパスを入力することで外部サイトにリダイレクトさせている。

hxxps://m.example.com//evil.com/%2F..

```http
HTTP/1.1 303 See Other
...
Location: //evil.com/%2F../
```

Bobrov氏はこの動作をNode.jsの[serve-staticの脆弱性](https://github.com/expressjs/serve-static/issues/26)（CVE-2015-1164）によるものと推測している。

#### [#387007](https://hackerone.com/reports/387007)

既存のパスのみトレイリングスラッシュを追加するリダイレクトが発生する場合もある。存在しないパスの `GET /xxx HTTP/1.1` は404だが、存在するパスの `GET /css HTTP/1.1` には `Location: /css/` がレスポンスされるような動作である。存在するパスでないとリダイレクトは発生しないため任意の文字列を挿入する余地はないように思えるが、`..;/` という一つ上の階層を示すような記号と組み合わせることで外部サイトのURLを挿入できたと考える。

hxxps://idp.fr.example.org//evil.com/..;/css

```http
HTTP/1.1 302 Found
...
Location: //evil.com/..;/css/
```

Bobrov氏はこの動作を自身が発見した[Apache Tomcatの脆弱性](https://tomcat.apache.org/security-9.html#Fixed_in_Apache_Tomcat_9.0.12)（CVE-2018-11784）によるものと推測している。

### クエリ文字列を追加するリダイレクトでの事例

#### [#192375](https://hackerone.com/reports/192375)

リクエストしたパスにクエリ文字列を追加するためのリダイレクトが発生する場合もあった。任意のパスに `?dmr_refresh=1` というクエリ文字列を追加し、相対パスでリダイレクトさせていたため外部サイトのURLを挿入できた。

hxxps://ml.money.example.jp//evil.com

```http
HTTP/1.1 302
...
Location: //evil.com?dmr_refresh=1
```

## Pythonに残るパスからのオープンリダイレクト

Pythonの標準HTTPモジュールには#125000と同様の脆弱性が報告されており、[issue32084](https://bugs.python.org/issue32084)で公開されている。2018年11月時点の最新バージョンである3.7.1および2.7.15では脆弱性は修正されていない。そのため手元のPythonで起動したWebサーバーはパスからのオープンリダイレクトに脆弱な状態である。Python3系では `python -m http.server` を、Python2系では `python -m SimpleHTTPServer` を実行し、以下のURLにアクセスすると外部サイトである `atsunoda.github.io` にリダイレクトされる。

http://127.0.0.1:8000//atsunoda.github.io/%2F%2E%2E

```http
HTTP/1.0 301 Moved Permanently
...
Location: //atsunoda.github.io/%2F%2E%2E/
```

## 所感

[Google](https://sites.google.com/site/bughunteruniversity/nonvuln/open-redirect)や[LINE](https://bugbounty.linecorp.com/ja/)などの一部企業を除いて<sup id="f3">[3](#fn3)</sup>、オープンリダイレクトは多くの報奨金制度で認定される脆弱性である。一般的なパラメータからの入力によるオープンリダイレクトは主要な脆弱性スキャナーで検査されている。しかし今回取り上げたパスからのオープンリダイレクトを検査する脆弱性スキャナーは少ないだろう。またトレイリングスラッシュに関するリダイレクトはサイト仕様を把握することなく見つけられる。脆弱性スキャンから漏れやすく、比較的容易に調査できる、狙い目の脆弱性といえる。フィッシングに悪用できる程度の脅威では高額な報奨金は望めないとわかっていても、探さずにはいられない。

---

<sup id="fn1">[1](#f1)</sup> 絶対パスで出力される場合のパスからのオープンリダイレクトは[#309058](https://hackerone.com/reports/309058)、[#211213](https://hackerone.com/reports/211213)、[#190188](https://hackerone.com/reports/190188)などが参考になる。  
<sup id="fn2">[2](#f2)</sup> 寺田さんのブログ「[オープンリダイレクト検査：Locationヘッダ編](http://d.hatena.ne.jp/teracc/20110212)」のパターン2でも解説されている。  
<sup id="fn3">[3](#f3)</sup> [Electronアプリでのオープンリダイレクト](https://blog.bentkowski.info/2018/07/vulnerability-in-hangouts-chat-aka-how.html)や[認証トークンの漏洩につながるケース](https://www.slideshare.net/linecorp/line-security-bug-bounty-program-97008237/28)には報奨金が支払われている。  
