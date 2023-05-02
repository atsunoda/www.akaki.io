---
description: 2018年5月5日に「モナバコ」というQ&Aサービスが脆弱性報奨金制度を導入した。国内の小規模サービスによるバグバウンティの独自開催は珍しく、モナコインという仮想通貨での報奨金支払いも異例だったため、興味をひかれ参加した。そこで認定された401インジェクションの脆弱性についてまとめる。
---

# モナバコ脆弱性報奨金制度で認定された401インジェクション

<p class="modest" align="left">May 17, 2018</p>

---

2018年5月5日に「[モナバコ](https://web.archive.org/web/20190526035228/https://monabako.com/#/)」というQ&Aサービスが[脆弱性報奨金制度](https://web.archive.org/web/20190526035228/https://monabako.com/#/bugbounty)を導入した。国内の小規模サービスによるバグバウンティの独自開催は珍しく、モナコインという仮想通貨での報奨金支払いも異例だったため、興味をひかれ参加した。そこで認定された401インジェクションの脆弱性についてまとめる。

## モナバコの概要

モナバコはTwitterを介したQ&Aサービスであり、Twitterアカウントでログインしてサービスを利用する。初回ログイン時に質問を受け付けるページが開設されるので、募集する質問内容をそこへ記入する。その後、質問ページを自身のTwitterで告知して質問を募る。質問ページを訪れたフォロワーはモナコインを付けて質問でき、それに回答した対価としてモナコインをもらえる仕組みである。

開設された質問ページにはアクセス制御が施されていないため、URLを知っていれば誰でもアクセスできる。脆弱性はこのページに存在していた。

<img src="/assets/2018/401i_in_monabako/monabako.webp" width="770" height="493" decoding="async" alt="monabako">

## 401インジェクションの脆弱性

画像などを表示する際のURLを任意の値に指定できる場合、Basic認証などが設定されたページのURLを挿入することで、ソース読み込み時に認証ダイアログが表示される問題が401インジェクションである。認証時のステータスコードである401が名前の由来になっている。この問題は一部のブラウザでのみ発生する挙動であり、今回はSafari、Edge、IE11で実証した。

モナバコのアカウントは、ログイン後に送信されるAPIリクエスト `postOauthToken` によって、Twitterアカウントと同様のプロフィール名とアイコン画像が設定される。その際、`photoURL` を以下のように操作することで任意のURLをアイコン画像のソースに指定できた。実証では私が所有するHeroku上にBasic認証を設定したページを用意し、そのURLを挿入した。

```diff
 POST /postOauthToken HTTP/1.1
 Host: api.monabako.com
 User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:59.0) Gecko/20100101 Firefox/59.0
 Accept: application/json, text/plain, */*
 Accept-Language: ja,en-US;q=0.7,en;q=0.3
 Accept-Encoding: gzip, deflate
 Referer: https://monabako.com/
 Content-Type: application/json;charset=utf-8
 token: eyJhb ... 4hUlQ
 Content-Length: 233
 Origin: https://monabako.com
 Connection: close

-{"token":"98 ... PM","secret":"xI ... sL","photoURL":"https://abs.twimg.com/sticky/default_profile_images/default_profile_normal.png","displayName":"Akaki Tsunoda"}
+{"token":"98 ... PM","secret":"xI ... sL","photoURL":"https://example.herokuapp.com/basic_auth.php","displayName":"Akaki Tsunoda"}
```

質問ページにはアイコン画像が設置されるため、一部のブラウザからアクセスすると認証ダイアログが表示される。以下はSafariからアクセスした際の様子である。

<img src="/assets/2018/401i_in_monabako/401i_dialog.webp" width="770" height="495" decoding="async" alt="401i_dialog">

imgタグのsrc属性にはBasic認証を設定したページのURLが挿入されている。

<img src="/assets/2018/401i_in_monabako/401i_src.webp" width="770" height="493" decoding="async" alt="401i_src">

攻撃者は不正なURLを仕込んだ質問ページをTwitterなどで拡散する。そこにアクセスした被害者が表示された認証タイアログをモナバコのログイン認証と誤認した場合、入力した認証情報が攻撃者の手に渡る恐れがあった。

## 脆弱性の修正漏れ
この脆弱性への対策として、Twitter社が所有するドメイン `twimg.com` のURLしか指定できないように修正された。しかしドメイン検証の実装に不備があったため、`twimg.com.example.com` のような任意のドメインのURLを指定できた。

<img src="/assets/2018/401i_in_monabako/401i_bypass.webp" width="770" height="493" decoding="async" alt="401i_bypass">

攻撃者は所有するドメインに2階層のサブドメイン `twimg.com.*` を設定することで、そのドメイン上のリソースを挿入できる状態だった。

## 所感

401インジェクションは[Twitterで認定された事例もある](https://hackerone.com/reports/221328)が、著名なバグハンターである[@EdOverflow](https://twitter.com/edoverflow)氏が公開する[セキュリティポリシー](https://github.com/EdOverflow/hackerone-security-policy/blob/master/POLICY.md)では「通常許容されるリスク」として認定外になっている。必ずしも認定される脆弱性ではないが、imgタグが読み込んだリソースをブラウザが処理する際にセキュリティリスクが生じる可能性もあるため、外部リソースを指定できる場合は401インジェクションを挙げて報告する価値はある。

今回発見した脆弱性は、制度に従いモナバコ公式Twitter（[@monabako](https://twitter.com/monabako)）へDMで報告した。軽微な脆弱性だったが変り種として認定してもらい、「Thank you」を意味すると思われる[39MONA](https://1manen.net/crypto.php?amount=39&currency=MONA)を頂いた<sup id="f1">[¹](#fn1)</sup>。認定の連絡から1時間もたたずに入金されたのには驚き、仮想通貨での報奨金支払いのメリットを実感した。

##### 時系列

2018年5月8日 - 401iを発見  
2018年5月8日 - 401iをモナバコへ報告  
2018年5月9日 - モナバコから認定および修正完了の連絡  
2018年5月9日 - モナバコから39MONAが入金  
2018年5月15日 - 修正漏れをモナバコへ報告  
2018年5月16日 - モナバコから修正完了の連絡  
2018年5月16日 - 修正完了を確認  

---

<sup id="fn1">[¹](#f1)</sup> 入金された2018年5月9日時点では、39MONAは日本円で19,110円（1MONA/490円換算）の価値があった。
