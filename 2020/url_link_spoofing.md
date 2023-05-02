---
description: フィッシング詐欺における偽サイトへの誘導では、正規サイトのURLをかたってリンクをクリックさせる手口が後を絶たない。この手口は主にメールや掲示板からの誘導で悪用されるが、グループチャット内でもURLリンクを偽装できれば脅威になると考えた。ビジネスチャットツールとして世界的に利用されているSlackでのURLリンク偽装（#481472）について解説する。
---

# Slackで認定されたURLリンク偽装の脆弱性

<p class="modest" align="left">Apr 27, 2020</p>

---

フィッシング詐欺における偽サイトへの誘導では、正規サイトのURLをかたってリンクをクリックさせる手口が後を絶たない。この手口は主にメールや掲示板からの誘導で悪用されるが、グループチャット内でもURLリンクを偽装できれば脅威になると考えた。ビジネスチャットツールとして世界的に利用されているSlackでのURLリンク偽装（[#481472](https://hackerone.com/reports/481472)）について解説する。

## URLリンクの偽装

HTMLでは、ハイパーリンクとして表示するURLと、実際にリンクするURLを個別に設定できる。例えば、以下のHTMLはレンダリングにより [`https://example.com`](https://akaki.io) となる。表示上は `example.com` へのリンクに見えるが、実際には `akaki.io` へリンクする。

```html
<a href="https://akaki.io">https://example.com</a>
```

このように、URLリンクを偽装する行為は「[Link Manipulation](https://www.phishing.org/phishing-techniques#:~:text=Link%20Manipulation)」というフィッシング手法として知られており、偽のログインページや正規ファイルに偽装したマルウェアなどへの誘導に悪用される。私宛に届いたAppleをかたるフィッシングメールでもURLリンクが偽装されていた。

<p align="center"><img src="/assets/2020/url_link_spoofing/apple_phishing.png" width="600" alt="apple_phishing"></p>

## Slackの脆弱性

脆弱性を発見した2019年1月当時、SlackのUIはハイパーリンクを投稿できない仕様であった。しかし、APIにはハイパーリンクを投稿するための `<{url}|{text}>` という記法が既に存在していた<sup id="f1">[¹](#fn1)</sup>。そこで、投稿時のHTTPリクエストを以下のように操作するとURLリンクを偽装できた。

```diff
 POST /api/chat.postMessage HTTP/1.1
 Host: █████████.slack.com
 ...

 ...
 -----------------------------87462859699239992111770463
 Content-Disposition: form-data; name="text"

-http://example.com
+<http://evil.com|http://example.com>
 -----------------------------87462859699239992111770463
 ...
```

<img src="/assets/2020/url_link_spoofing/spoof_url.png" width="600" alt="spoof_url">

Slackのワークスペースでは同じ名前のメンバーが共存できる<sup id="f2">[²](#fn2)</sup>。そのため、ワークスペース内で影響力のある人物になりすまして偽装したURLリンクを投稿することで、他のメンバーからのクリック率を高められると考えた。レポートでは、FacebookのCEOであるMark Zuckerberg氏が参加するワークスペースを仮定し、求人募集をうたう投稿の例を示した。

<img src="/assets/2020/url_link_spoofing/fake_url.png" width="800" alt="fake_url">

Slackは当初、「この攻撃はワークスペース内でのメンバー間の対立が前提になっている」と述べ、URLリンクの偽装を脆弱性として認めなかった。その後、私はSlackとOutlookを連携できることを知り<sup id="f3">[³](#fn3)</sup>、もしメンバーのひとりがOutlookで受信したメールをSlackに転送していた場合、攻撃者はそのメンバー宛にメールを送り付けることで、外部から偽装したURLリンクを投稿できると考えた。「そのような攻撃はメンバー間の対立とは限らない」と再検討を依頼すると、Slackはこの偽装を脆弱性として認め対策を講じた。

### 脆弱性対策とその回避

脆弱性への一次対応として、SlackはURLリンクを含むメッセージをJSON形式で送信する仕様に変更した。この形式で送られるURLに改行文字 `\n` を含めると、投稿後のURLリンクが2行に分割された。そのため、URLのuserinfo要素（ `@` より前）に偽のホスト名と複数の改行文字を加えることで、以下のような偽装が可能となった。

```diff
 POST /api/chat.postMessage HTTP/1.1
 Host: █████████.slack.com
 ...

 ...
 -----------------------------7333129693824198502059347126
 Content-Disposition: form-data; name="blocks"

-[{"type":"rich_text","elements":[{"type":"rich_text_section","elements":[{"type":"link","url":"https://example.com"}]}]}]
+[{"type":"rich_text","elements":[{"type":"rich_text_section","elements":[{"type":"link","url":"https://example.com\n\n\n\n\n@evil.com"}]}]}]
 -----------------------------7333129693824198502059347126
 ...
```

<img src="/assets/2020/url_link_spoofing/userinfo_url.png" width="600" alt="userinfo_url">

### 脆弱性への緩和策

2020年2月にSlackは、UIからハイパーリンクを投稿できる機能をリリースした<sup id="f4">[⁴](#fn4)</sup>。書式設定のリンクアイコン `🔗` から任意のリンクを生成できるため、実際のリンク先と表示が異なるURLリンクの投稿も可能である。しかし、そのように偽装されたURLリンクをクリックすると、以下のような警告ダイアログが現れるようになった。

<p align="center"><img src="/assets/2020/url_link_spoofing/warning_dialog.png" width="600" alt="warning_dialog"></p>

さらに、リンクにカーソルを合わせると、URLがポップアップで表示されるようにもなった。これにより、ステータスバーが無いデスクトップアプリでも実際のリンク先を確認できる。改行文字による偽装もポップアップには及ばない。

<img src="/assets/2020/url_link_spoofing/link_popup.png" width="600" alt="link_popup">

## 所感

ハイパーリンクを投稿できるチャットではURLリンクの偽装が可能となる。知り合いだけのクローズドなコミュニティであれば、悪用はメンバー同士のトラブルなどが前提となるため、当初のSlackのように運営者がリスクを受容するのも理解できる。しかし、誰でも参加できるオープンなコミュニティでは、攻撃者が紛れ込む事態も想定される。URLリンクを踏むたびに偽装を疑うのは煩わしく、また全ての利用者が偽装を見破れるわけではないため、ダイアログによる警告は意図しないアクセスの防止策として効果が期待できる。利用者による真偽の見極めだけに委ねずに、運営者として対策に取り組んだSlackの姿勢は好感が持てる。

Slackに限らず、サイトやアプリの至る所にあるURLリンクを私は日々クリックし、タップしている。偽装の事例を知ってからは、アクセスする前に実際のリンク先を気にするようになった。主要なブラウザやメーラーでは、リンクにカーソルを合わせることでステータスバーからURLを確認できる。モバイルアプリも多くの場合、リンクの長押しによりURLを確認できる。ただし、JavaScriptによる遷移先までは確認できないため、攻撃者の意図したスクリプトが動作する場合は、ステータスバーの表示も偽装されている可能性がある<sup id="f5">[⁵](#fn5)</sup>。信用できない環境ではリンクを踏まないという判断も必要だ。

---

<sup id="fn1">[¹](#f1)</sup> https://api.slack.com/reference/surfaces/formatting#linking-urls  
<sup id="fn2">[²](#f2)</sup> https://twitter.com/buritica/status/970721576034455552  
<sup id="fn3">[³](#f3)</sup> https://slackhq.com/increase-everyday-productivity-with-office-365-apps-for-slack  
<sup id="fn4">[⁴](#f4)</sup> https://twitter.com/SlackHQ/status/1230587660948762624  
<sup id="fn5">[⁵](#f5)</sup> <a href="https://www.owenboswarva.com/URLspoof.htm" onclick="this.href='https://evil.akaki.io'">https://www.owenboswarva.com/URLspoof.htm</a>
