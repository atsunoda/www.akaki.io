---
lang: ja
---

# Yahoo! JAPAN IDに登録された電話番号の列挙

<time datetime="2020-02-03">2020年2月3日</time>

---

Webサービスのアカウントに登録されている携帯電話番号を特定できれば、そのサービスの利用者を標的としたスミッシング（SMSフィッシング）が可能になると考えた。日本最大級のポータルサイトである「Yahoo! JAPAN」のアカウントリカバリー機能では、Yahoo! JAPAN IDに登録した電話番号からアカウントの存在を確認できる。この機能の挙動を利用して他人のIDに紐づく電話番号を列挙できる状態だった。発見した脆弱性は[YJ-CSIRT](https://www.nca.gr.jp/member/yj-csirt.html)へ報告し、既に修正されている。

## ユーザー列挙の脆弱性

入力値の違いにより生じるエラーメッセージや応答時間などの差異を手がかりに、アカウントの識別情報を収集する行為を「ユーザー列挙（User Enumeration）」という。例えばアカウントを登録する際に以下のようなエラーが発生した場合、入力した値は既に他の利用者がユーザー名として使用していることが分かる。

<figure><img src="/assets/2020/phone_number_enumeration/google_signup.webp" width="500" height="142" decoding="async" alt="" /></figure>

認証情報となるユーザー名を大量に収集できれば、攻撃者は逆総当たり攻撃によるアカウントの乗っ取りを試みる。メールアドレスなどの連絡先情報を収集できればフィッシング詐欺にも悪用する。脆弱性となり得る挙動を[OWASP Web Security Testing Guide](https://github.com/OWASP/wstg/blob/master/document/4_Web_Application_Security_Testing/4.4_Identity_Management_Testing/4.4.4_Testing_for_Account_Enumeration_and_Guessable_User_Account_OTG-IDENT-004.md)が解説している。

## IDに紐づく電話番号の列挙

Yahoo! JAPANのアカウントリカバリー機能は、Yahoo! JAPAN IDやそれに紐づくメールアドレスの他に、SMS認証に設定した携帯電話番号からもアカウントの存在を確認できる仕様となっている。発見した脆弱性により他人のIDやメールアドレスも列挙できる状態だったが、私は電話番号の列挙だけを実証した。

### 紐づきを特定できる挙動

アカウントリカバリー機能には機械的な試行を防止するCAPTCHA認証が搭載されている。私のYahoo! JAPAN IDに紐づけた電話番号 `080██████03` と正しいCAPTCHA文字を以下のように入力すると、次の本人確認フェーズに進める。

<figure><img src="/assets/2020/phone_number_enumeration/yj_ar.webp" width="770" height="578" decoding="async" alt="" /></figure>

しかし、私の電話番号の下1桁を変更した `080██████04` と正しいCAPTCHA文字を入力すると、以下のようなエラーが発生する。

<figure><img src="/assets/2020/phone_number_enumeration/yj_ar_err.webp" width="770" height="600" decoding="async" alt="" /></figure>

エラーメッセージの内容から、電話番号 `080██████04` に紐づくIDは存在しないことが分かる。IDに紐づく電話番号とそうでない番号を入力した場合に生じる挙動の差異をもとに、他人のIDに紐づく電話番号を特定できる。

### 機械的な試行による列挙

この挙動を利用して大量の電話番号を列挙するには機械的な試行が必要になる。CAPTCHA認証が搭載されているため困難に思えるが、認証の成否に関わらずIDとの紐づきを判定していたため機械的な列挙が可能であった。

例えば、私の電話番号と誤ったCAPTCHA文字 `あ` を入力すると、再認証を求める旨のエラーが発生する。

<figure><img src="/assets/2020/phone_number_enumeration/yj_captcha.webp" width="770" height="584" decoding="async" alt="" /></figure>

しかし、IDに紐づかない電話番号と誤ったCAPTCHA文字を入力すると、再認証を求める旨のエラーだけでなく、電話番号に紐づくIDが存在しない旨のエラーも発生する。

<figure><img src="/assets/2020/phone_number_enumeration/yj_captcha_err.webp" width="770" height="620" decoding="async" alt="" /></figure>

CAPTCHA認証に失敗しても電話番号とIDの紐づきを確認できる。そこで私の電話番号の下1桁をBurp SuiteのIntruderで総当たりすると、他人のIDに紐づく5つの電話番号を列挙できた。電話番号がIDに紐づく場合のレスポンスサイズは約16500バイトであるのに対して、紐づかない場合はエラーメッセージ分が増加して約17000バイトになっている。

<figure><img src="/assets/2020/phone_number_enumeration/burp_enum.webp" width="770" height="513" decoding="async" alt="" /></figure>

列挙できた電話番号の1つである `080██████09` を用いてアカウントリカバリー機能を実行すると、私の所有ではないメールアドレスの一部が表示されることから、他人のIDに紐づく電話番号であると判断した。

<figure><img src="/assets/2020/phone_number_enumeration/yj_ar_email.webp" width="770" height="435" decoding="async" alt="" /></figure>

## フィッシング詐欺への発展

他人のIDに紐づく電話番号 `080██████09` は、Yahoo! JAPANの利用者が所有する携帯電話番号でもある。そのためYahoo! JAPANをかたるスミッシングの標的になり得る。巧妙な攻撃者は被害者を動揺させ、判断力を鈍らせてから詐欺をしかけてくる。一案として、攻撃者はフィッシングメッセージを送り付ける前に、Yahoo! JAPANのSMS認証設定などを利用して正規のSMSを被害者へ何通か送り付ける。

<figure><img src="/assets/2020/phone_number_enumeration/yj_sms.webp" width="300" height="307" decoding="async" alt="" /></figure>

被害者は身に覚えのないSMS認証の試行に気づき、自分のアカウントが被害にあっているのではないかと不安になる。そこへ攻撃者はYahoo! JAPANをかたり、被害者の動揺を誘う以下のようなSMSを送り付ける。正規のSMSとは異なるスレッドに着信するが、送信者IDは `Yahoo JAPAN` であり、文面にはYahoo! JAPAN IDに登録したメールアドレスの一部も含まれるため、動揺した被害者はフィッシングメッセージの内容を信じてしまう。

<figure><img src="/assets/2020/phone_number_enumeration/yj_sms_spoof.webp" width="300" height="326" decoding="async" alt="" /></figure>

## 所感

携帯電話番号は桁数の決まった数字のみで構成され規則性もあるため、メールアドレスに比べて総当たりが容易である。私の電話番号の下2桁を総当たりしていれば、より多くの利用者の電話番号を列挙できただろう。脆弱性は報告した3週間後に修正され、CAPTCHA認証に成功しないとIDとの紐づきを確認できなくなった。しかし紐づきの有無による挙動の差異は現在も生じるため、日本語に対応した自動認識技術や人力解読サービスを用いてCAPTCHA認証を回避できれば、一定量の列挙は可能だと考える。

その他の機能でもIDに紐づく電話番号を特定できる挙動は見受けられるが、Yahoo! JAPANではCAPTCHA認証やレート制限によって収集行為を緩和している。ユーザビリティの優先度や修正の費用対効果を理由にユーザー列挙の脆弱性を受容するケースは少なくない。[Facebook](https://averagesecurityguy.github.io/2016/09/07/facebook-private-phone-enumeration/)や[Uber](https://hackerone.com/reports/138881)でも電話番号を列挙できる挙動は見つかっているが、両社ともレート制限による緩和策を講じていたため脆弱性として認定していない。利用者の電話番号を攻撃者に特定された場合、SMS経由のフィッシング詐欺だけでなく、通話による架空請求詐欺などにも悪用される恐れがある。電話番号を特定できる挙動を仕様とする場合は、収集行為への厳しい緩和策が求められる。
