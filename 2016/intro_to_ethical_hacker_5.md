# 脆弱性を見つけよう クロスサイトスクリプティング編

<p class="modest" align="left">Dec 28, 2016</p>

---

第5回目からはWebアプリケーションの様々な脆弱性の見つけ方を紹介していきます。

各脆弱性の詳しい説明や脅威、対策方法については、IPAから提供されている「[安全なウェブサイトの作り方](https://www.ipa.go.jp/security/vuln/websecurity.html)」を参照ください。また、以下の書籍も参考になります。

『[体系的に学ぶ 安全なWebアプリケーションの作り方](https://www.sbcr.jp/product/4797361193/)』SBクリエイティブ

## 検証環境の構築

Webアプリケーションセキュリティの普及・啓発活動を行なう非営利団体「[Open Web Application Security Project](https://owasp.org/)（OWASP）」から、脆弱性の検証や学習向けに仮想環境「OWASP BWA」が提供されています。これには脆弱性が作り込まれたWebアプリケーションがいくつも盛り込まれています。ローカルにこの環境を構築し、様々な脆弱性を実際に見つけていきましょう。

OWASP BWAのセットアップは簡単です。[OWASP Broken Web Applications Project](https://web.archive.org/web/20161010022044/https://www.owasp.org/index.php/OWASP_Broken_Web_Applications_Project)のリンクからovaファイルをダウンロードして、VMwareやVirtualBoxなどの仮想マシンから開くだけです。図1ではVMwareから仮想環境を起動しています。起動後に表示されるURLをブラウザから開くことでOWASP BWAにアクセスできます。

<p align="center"><img src="/assets/2016/intro_to_ethical_hacker_5/e5_figure1.png" alt="figure1"></p>
<p class="modest" align="center">図1. OWASP BWAを起動してブラウザからアクセス</p>

今回はOWASP BWAの中から「BodgeIt」というサイトを対象に調査を行ないます。このサイトでは課題が用意されていて、脆弱性を用いて課題をクリアするとスコアを獲得できます。

<p align="center"><img src="/assets/2016/intro_to_ethical_hacker_5/e5_figure2.png" alt="figure2"></p>
<p class="modest" align="center">図2. 課題をクリアするとスコアが緑に変わる</p>

## 検出方法

今回はクロスサイトスクリプティングの脆弱性を取り上げます。クロスサイトスクリプティング（XSS）とは、サイト上で入力値をHTMLやJavaScriptなどへ不適切に出力していた場合、攻撃者の仕掛けたスクリプトがサイト利用者のブラウザ上で実行されてしまう問題です。攻撃手法によってReflected XSS、Stored XSS、DOM Based XSSに分類されます。まずはリクエストに含まれる値がレスポンスに出力される箇所で起こるReflected XSSを見つけましょう。

Reflected XSSを見つけるには以下の手順でサイトの挙動を確認します。

1. 入力値がレスポンスに出力されるか
2. 特別な意味を持つ記号がエスケープされずに出力されるか
3. サイト上で任意のスクリプトを実行できるか

一部のブラウザではXSSから利用者を保護する機能が内蔵されていて、スクリプトを実行できない場合があります。ここでは保護機能が内蔵されていないFirefoxを使ってサイトの挙動を確認します。

BodgeItのページや機能を見ていくと、検索ページで検索したワードが画面に表示されています。レスポンスを見るとHTMLタグの間に出力されることが分かります。

<p align="center"><img src="/assets/2016/intro_to_ethical_hacker_5/e5_figure3.png" alt="figure3"></p>
<p class="modest" align="center">図3.「test」と検索した際の挙動</p>

入力値が出力される箇所を見つけたら、記号がエスケープされるか確かめます。HTMLタグの間に出力されるような場合は、適当なHTMLタグを入力して表示のされ方を確認しましょう。`<s>test</s>` と入力して検索すると、画面には「~~test~~」のように線が引かれて表示されました。

<p align="center"><img src="/assets/2016/intro_to_ethical_hacker_5/e5_figure4.png" alt="figure4"></p>
<p class="modest" align="center">図4.「&lt;s&gt;test&lt;/s&gt;」と検索した際の挙動</p>

この線は `<s>` タグによる効果です。検索ワードに含まれた記号はエスケープされず、HTMLタグとして有効な状態で出力されています。`<script>` タグを使ってスクリプトも実行できそうです。BodgeItでは `<script>alert("XSS")</script>` というパターンでスクリプトを実行することで、XSSの脆弱性を見つけたことになります。このパターンを入力すると `alert()` メソッドが動作して、画面にポップアップが表示されると思います。課題はクリアとなりスコアも変わるはずです。

## 検出パターン

検索ページの他に、お問い合わせページでも入力した内容が画面に表示されています。`<s>` タグも有効に出力されたので、`<script>alert("XSS")</script>` を入力したところ図5のように出力されました。

<p align="center"><img src="/assets/2016/intro_to_ethical_hacker_5/e5_figure5.png" alt="figure5"></p>
<p class="modest" align="center">図5.「&lt;script&gt;alert("XSS")&lt;/script&gt;」と入力した際の挙動</p>

これではスクリプトを実行できません。`<script>` や `"` など、特定の文字列や記号を出力しないことでXSS対策を講じているようです。しかし、このようなブラックリスト方式のXSS対策は実装が不十分になりやすく、回避できる可能性が高いです。このような対策の回避方法は、OWASPの「[XSS Filter Evasion Cheat Sheet](https://web.archive.org/web/20161020134045/https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet)」にまとめられています。また、JPCERT/CC（一般社団法人 JPCERTコーディネーションセンター）から[日本語訳](https://jpcertcc.github.io/OWASPdocuments/CheatSheets/XSSFilterEvasion.html)も公開されています。

このチートシートにはスクリプトを実行できる様々なパターンが紹介されています。`<script>` タグを使わないパターンなどを入力することで、お問い合わせページでもスクリプトを実行できると思います。しかしスコアは変わらないので、他にもXSSの脆弱性があるようです。サイト内にはアカウント登録ページもあるので、登録した値がレスポンスに出力される箇所で起こるStored XSSも狙ってみてください。

<br>

BodgeItでは `alert("XSS")` というスクリプトを実行することでクリアとなりますが、実際のバグバウンティでXSSの脆弱性を報告する際はもう少し確認しましょう。例えば `alert(document.domain)` を実行して、スクリプトが対象のサイト上（[オリジン](https://ja.wikipedia.org/wiki/%E5%90%8C%E4%B8%80%E7%94%9F%E6%88%90%E5%85%83%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC#Origin.E5.88.A4.E5.AE.9A.E3.83.AB.E3.83.BC.E3.83.AB)）で実行されることを確かめましょう。また、脆弱性を悪用してサイト利用者に不正なスクリプトを実行させる方法も考えてみてください。

次回も一つ脆弱性を取り上げ、その見つけ方を紹介します。良いお年を。
