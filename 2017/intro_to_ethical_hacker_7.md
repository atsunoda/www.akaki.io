---
description: 第7回目はSQLインジェクションの脆弱性を取り上げます。
---

# 脆弱性を見つけよう SQLインジェクション編

<p class="modest" align="left">Mar 27, 2017</p>

---

第7回目はSQLインジェクションの脆弱性を取り上げます。

SQLインジェクションとは、データベースを使用しているサイトで入力値をSQLの一部として不適切に組み込んでいた場合、攻撃者の意図したSQLを発行されてしまう問題です。データベースに保存した情報を攻撃者に盗まれたり、改ざんされたり、認証を回避される恐れがあります。以前に取り上げたXSSやCSRFとは異なり、サイト自体への直接攻撃に悪用される脆弱性です。

## 検出方法

SQLインジェクションを見つけるには以下の手順でサイトの挙動を確認します。

1. 入力値をSQLに使用していそうな機能があるか
2. SQL構文の異常系と正常系を入力した際に挙動が変化するか、またSQLエラーが表示するか
   * 文字列型の場合は `'` と `''` を追加した際の変化など
   * 数値型の場合は `*1()` と `*(1)` を追加した際の変化など
3. SQL特有の句や式、関数が使用できるか、またミスタイプで挙動が変化するか
   * 文字列型の場合は `' and '1' = '1` と `' axd '1' = '1` を追加した際の変化など
   * 数値型の場合は `*(select 1)` と `*(sxlect 1)` を追加した際の変化など
4. データベースの情報を取得できるか

今回はOWASP BWAの「WackoPicko」というサイトを対象に調査します。このサイトにはログイン機能があり、デフォルトユーザー「bryce」が存在します。このユーザーは[Joeアカウント](https://web.archive.org/web/20170318231444/http://securityblog.jp/words/joe_account.html)です。ログインに成功するとリダイレクトが発生し、アカウントのHomeページに遷移しました。

<p align="center"><img src="/assets/2017/intro_to_ethical_hacker_7/e7_figure1.png" alt="figure1"></p>
<p class="modest" align="center">図1. ログイン成功後のページ</p>

ログイン機能ではデータベースに保存された情報をもとに認証していそうです。Burp SuiteのRepeater機能を使って、ログイン時のPOSTリクエストを操作してみましょう。パラメータ `username` の値にシングルクォーテーション `'` を追加すると以下のようなメッセージが表示されました。

<p align="center"><img src="/assets/2017/intro_to_ethical_hacker_7/e7_figure2.png" alt="figure2"></p>
<p class="modest" align="center">図2. シングルクォーテーションを追加した際の挙動</p>

これはSQLの構文エラーが発生した際に出力されるエラーメッセージのようです。2つのシングルクォーテーション `''` を追加するとSQLの構文エラーは発生しません。この機能にはSQLインジェクションの脆弱性がありそうです。SQL特有の句や式が使えるか確認しましょう。

ここでSQLインジェクションの解説でよく用いられる `' or 1 = 1 --` を試さないようにしましょう。その理由については[とある診断員氏のスライド](https://www.slideshare.net/zaki4649/sql-35102177/33)をご覧ください。OR演算子や行コメントを用いるとデータベースの情報を壊わしてしまう恐れがあるので要注意です。

AND演算子を用いた `' and '1' = '1` を追加するとリダイレクトが発生し、ユーザー「bryce」としてログインに成功しました。

<p align="center"><img src="/assets/2017/intro_to_ethical_hacker_7/e7_figure3.png" alt="figure3"></p>
<p class="modest" align="center">図3.「' and '1' = '1」を追加した際の挙動</p>

また、AND演算子をタイポした `' axd '1' = '1` を追加するとSQLエラーが発生しました。

<p align="center"><img src="/assets/2017/intro_to_ethical_hacker_7/e7_figure4.png" alt="figure4"></p>
<p class="modest" align="center">図4.「' axd '1' = '1」を追加した際の挙動</p>

どうやらパラメータ `username` の値はSQLに組み込まれているようです。SQLインジェクションによってデータベースの情報を取得できることを確認しましょう。

## 実証

データベースには他人のアカウント情報や管理者情報など、そのサイトに関する重要な情報が保存されていることが予想されます。SQLインジェクションの脆弱性を確認する目的であれば、そのような情報を取得する必要はありません。ここではデータベースのバージョン情報を取得します。

バージョン情報に限らず、情報の保存場所や取得方法はデータベースの種類によって異なります。そのため、まずはデータベースの種類を特定する必要があります。今回はSQLエラーの内容に「MySQL」と表記されているため特定するのは簡単です。しかし、他のサイトでは「エラーが発生しました」のような単一メッセージしか表示されない場合もあります。そのような場合は[OWASP Backend Security Project](https://web.archive.org/web/20160915095249/https://www.owasp.org/index.php/OWASP_Backend_Security_Project_DBMS_Fingerprint#Fingerprinting_with_string_concatenation)の資料を参考に、文字列連結演算子を用いてデータベースを特定しましょう。

データベースがMySQLで、かつSQLエラーが出力している場合は、`extractvalue()` 関数を用いてエラーメッセージに取得したい情報を含めることができます。パラメータから`' and extractvalue(0,concat(0x0a,version())) = '` を入力すると、バージョン情報を含んだメッセージが表示されました。

<p align="center"><img src="/assets/2017/intro_to_ethical_hacker_7/e7_figure5.png" alt="figure5"></p>
<p class="modest" align="center">図5. extractvalue()関数を用いた際の挙動</p>

`extractvalue()` 関数の第二引数は[XPath](https://ja.wikipedia.org/wiki/XML_Path_Language)を想定しています。しかし、`version()` 関数の実行結果が渡されたことで、XPathの構文エラーが発生したのです。「この値はXPathじゃないよ」というエラーメッセージの「この値」の部分に `version()` 関数の実行結果、すなわちバージョン情報が含まれる仕組みです。

データベースの情報を取得できたことから、ログイン機能にSQLインジェクションの脆弱性があることは間違いないでしょう。

<br>

今回はこのような方法で情報を取得できましたが、もし他のデータベースが使用されていたら？SQLエラーが表示されなかったら？どのように取得できるでしょうか。例えば、[ブラインドSQLインジェクション](https://web.archive.org/web/20161109173408/https://www.owasp.org/index.php/Blind_SQL_Injection)で取得できるかもしれません。この方法での情報取得にも挑戦してみてください。

次回も一つ脆弱性を取り上げ、その見つけ方を紹介します。
