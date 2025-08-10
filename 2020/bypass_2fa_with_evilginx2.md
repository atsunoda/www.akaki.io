---
description: 国内のオンラインサービスになりすまして正規の2要素認証を回避するフィッシングサイトが確認されている。このようなフィッシングサイトは被害者と正規サイトの間に介入し、被害者によって入力された認証情報やワンタイムパスワード（OTP）を即時に正規サイトへ入力することで認証を回避する。MITM（Man-in-the-middle）フィッシングと呼ばれるこの手法を理解するため、フィッシングフレームワーク「Evilginx2」を使用して2要素認証の回避を検証した。
---

# MITMフィッシングによる2要素認証の回避

<time datetime="2020-11-09">Nov 9, 2020</time>

---

国内のオンラインサービスになりすまして正規の2要素認証を回避するフィッシングサイトが確認されている<sup id="f1">[¹](#fn1)</sup>。このようなフィッシングサイトは被害者と正規サイトの間に介入し、被害者によって入力された認証情報やワンタイムパスワード（OTP）を即時に正規サイトへ入力することで認証を回避する。MITM（Man-in-the-middle）フィッシングと呼ばれるこの手法を理解するため、フィッシングフレームワーク「Evilginx2」を使用して2要素認証の回避を検証した。

## MITMフィッシングの仕組み

OTPによる2要素認証はパスワード認証の補強手段として活用されているが、パスワードと同様にOTPも被害者によってフィッシングサイトに入力されてしまえば攻撃者の手に渡る。つまり、攻撃者は被害者と正規サイトの間に介入し、被害者になりすまして正規の認証手順を踏むことで正規サイトの認証を回避できる。実際に、MITM（中間者）攻撃を組み合わせたフィッシング詐欺によって、国内で広く利用されるメッセージングサービス「LINE」のログイン認証を回避されてアカウントを乗っ取られる被害や<sup id="f2">[²](#fn2)</sup>、インターネットバンキングの取引認証を回避されて預金を不正送金される被害が発生している<sup id="f3">[³](#fn3)</sup>。

このようなMITMフィッシングによって、Webサービスにログインする際の2要素認証としてのSMS認証が回避される手順を図1に示す。まず、攻撃者は正規サイトの運営者などをかたり、攻撃者サイトのURLを含めたフィッシングメッセージをSMSなどで被害者へ送信し、正規サイトに偽装した攻撃者サイトへログインするよう被害者を誘導する（手順1）。つぎに、被害者は受信したフィッシングメッセージに従い、正規サイトのIDとパスワードを攻撃者サイトに入力する（手順2）。その後、攻撃者サイトは2要素認証としてのSMS OTPの入力を被害者に要求する。それと並行して、攻撃者は窃取した被害者のIDとパスワードを正規サイトに入力する（手順3）。正規サイトは被害者の携帯電話番号へSMS OTPを送信する（手順4）。被害者は受信したSMS OTPを攻撃者サイトに入力する（手順5）。攻撃者は窃取した被害者のSMS OTPを正規サイトに入力する（手順6）。正規サイトは入力されたSMS OTPが被害者に送信した値と一致することから、攻撃者を被害者として認証する（手順7）。以上の手順により、攻撃者は被害者になりすまして正規サイトにログインできる。

<figure><img src="/assets/2020/bypass_2fa_with_evilginx2/sms-auth-bypass.webp" width="770" height="351" decoding="async" alt="" /><figcaption>図1. MITMフィッシングによるSMS認証回避の手順</figcaption></figure>

## Evilginx2を使用した2要素認証の回避

このようなMITMフィッシングの動作を検証するため、MITMフィッシングフレームワークである「[Evilginx2](https://github.com/kgretzky/evilginx2)」を使用して検証サイトを構築した。Evilginx2は被害者のブラウザと正規サイトの間でリバースプロキシとして動作し、正規サイトになりすまして認証情報やセッショントークンをキャプチャする<sup id="f4">[⁴](#fn4)</sup>。つまり、図1に示す攻撃者サイトの役割をEvilginx2が担う。同様のツールとして「[Modlishka](https://github.com/drk1wi/Modlishka)」や「[Muraena](https://github.com/muraenateam/muraena)」も知られているが、構築の容易性や更新状況を考慮してEvilginx2を選択した。検証では国内で広く利用されるキャッシュレス決済サービス「PayPay」を正規サイトと仮定し、AWSのEC2で動作するEvilginx2に複製させた。その結果、EC2上の検証サイトはPayPayのログインページを完全に複製し（図2）、そこへ入力したPayPayアカウントの携帯電話番号とパスワードはEvilginx2にキャプチャされた（図3）。なお、EC2へのインバウンド接続は私のIPアドレスのみを許可し、一般利用者が誤って検証サイトにアクセスしないよう制御した。

<figure><img src="/assets/2020/bypass_2fa_with_evilginx2/sp_fake_paypay.webp" width="300" height="534" decoding="async" alt="" /><figcaption>図2. Evilginx2に複製されたログインページ</figcaption></figure>

<figure><img src="/assets/2020/bypass_2fa_with_evilginx2/evilginx2_pw.webp" width="770" height="140" decoding="async" alt="" /><figcaption>図3. Evilginx2にキャプチャされた認証情報</figcaption></figure>

Evilginx2は正規サイトとのセッションを維持することから、パスワード認証後の2要素認証も回避できた。PayPayのパスワード認証に成功すると、SMS OTPによる2要素認証ページも複製された（図4）。その後、PayPayから送信されたSMS OTP（図5）を入力すると、その値もEvilginx2にキャプチャされ（図6）、ログイン成功時に発行されるセッショントークンもログに記録された。このセッショントークンの有効性はPayPayのブラウザ支払いから確認した。

<figure><img src="/assets/2020/bypass_2fa_with_evilginx2/sp_fake_paypay_form.webp" width="300" height="534" decoding="async" alt="" /><figcaption>図4. Evilginx2に複製された2要素認証ページ</figcaption></figure>

<figure><img src="/assets/2020/bypass_2fa_with_evilginx2/sp_fake_paypay_sms.webp" width="300" height="208" decoding="async" alt="" /><figcaption>図5. PayPayから送信されたSMS OTP</figcaption></figure>

<figure><img src="/assets/2020/bypass_2fa_with_evilginx2/evilginx2_otp.webp" width="770" height="106" decoding="async" alt="" /><figcaption>図6. Evilginx2にキャプチャされたOTP</figcaption></figure>

## 現実的な詐欺シナリオと対策

詐欺師はメールやSMSで被害者にリンクを送りつけ、正当な理由をかたってフィッシングサイトへ誘導する。例えば、詐欺師はPayPayの運営になりすまして「あなたのアカウントが不正利用されている」という主旨のSMSを送りつけ、被害者にパスワードを変更するようリンクからのログインを促す（図7）。被害者がフィッシングサイトに認証情報を入力するとPayPayからSMS OTPが送られてくる。iOSではSMSパスコード自動入力がフィッシングサイトでも機能することから、被害者はOTPを容易に入力できてしまう。PayPayから送られてくるSMSメッセージの送信元が `PayPay` と表記される場合、詐欺師は送信者IDを偽装することで正規メッセージと同じスレッドにフィッシングメッセージを混入できる<sup id="f5">[⁵](#fn5)</sup>。検証ではPayPayを正規サイトと仮定したが、詐欺師はあらゆるサービスになりすましてフィッシングを仕掛けてくる。

<figure><video controls muted playsinline poster="/assets/2020/bypass_2fa_with_evilginx2/sp_fake_paypay_poster.webp" src="/assets/2020/bypass_2fa_with_evilginx2/sp_fake_paypay.mp4" type="video/mp4" width="300"></video><figcaption>図7. PayPayをかたるフィッシング詐欺シナリオ</figcaption></figure>

このようなフィッシング詐欺に対して利用者ができる対策のひとつは、身に覚えのないメールやSMSで送られてきたリンクにアクセスしないことである。メールやSMSの内容が不正利用疑いのような不安に感じる警告であったり、キャッシュバックのような魅力的な告知であっても、反射的にリンクにアクセスしない。アカウントの状態を確認する場合は、リンクからではなく公式アプリからのログインを徹底する。また、フィッシング詐欺の手口はログインを求めるだけでなく、個人情報やクレジットカード情報の入力や、添付ファイルの開封やアプリのインストールを求める手口も確認されている。身に覚えがなければ無視したり、判断に迷うようであれば公式サポートに問い合わせたりすることも被害の防止につながる。

## 所感

MITMフィッシングへの対策は利用者の意識に頼るだけでなく、事業者によるFIDO認証の導入といった抜本策も必要だと考える。昨今のフィッシング詐欺は正規サイトの完全な複製にとどまらず、メッセージの送信元やリンクも巧妙に偽装する。そのため利用者がフィッシングを見抜くのは困難になりつつある。そこで、フィッシングによる認証情報の盗用を防止する策として「FIDO（Fast IDentity Online）」が注目されている。FIDOは、パスワードのような認証情報をサーバーへ送信して本人認証するのではなく、顔や指紋といった生体情報や所持するデバイスに基づいてクライアントで本人認証し、認証結果のみをサーバーへ送信する認証方式である。クライアントでの本人認証の前にサーバーの正当性を検証するため、フィッシングサイトでは本人認証自体が開始されない。FIDOの先駆けとして、2017年にGoogleは全ての従業員の認証にU2Fデバイスを導入することで、その年のフィッシングによるアカウントの乗っ取り被害を0件に抑えた<sup id="f6">[⁶](#fn6)</sup>。また、FIDO2の導入によりEvilginx2への耐性を実証した例もある<sup id="f7">[⁷](#fn7)</sup>。今後のFIDOの普及と、それによるフィッシング詐欺被害の減少に期待したい。

---

<sup id="fn1">[¹](#f1)</sup> [LINE Phishing Scam steals your SMS authentication code - ozuma5119](https://medium.com/@ozuma5119/line-phishing-scam-steals-your-sms-authentication-code-ee8b83585b81)  
<sup id="fn2">[²](#f2)</sup> [LINEへの不正ログインに対する注意喚起 - LINE Corporation](https://linecorp.com/ja/security/article/251)  
<sup id="fn3">[³](#f3)</sup> [ネットバンキング被害4倍に　「ワンタイムパス」破る - 日本経済新聞](https://www.nikkei.com/article/DGXMZO55313840W0A200C2MM0000/)  
<sup id="fn4">[⁴](#f4)</sup> [Evilginx 2 - Next Generation of Phishing 2FA Tokens](https://breakdev.org/evilginx-2-next-generation-of-phishing-2fa-tokens/)  
<sup id="fn5">[⁵](#f5)</sup> [SMSで送信元を偽装したメッセージを送る - Akaki I/O](/2019/sms_spoofing.md)  
<sup id="fn6">[⁶](#f6)</sup> [Google: Security Keys Neutralized Employee Phishing - Krebs on Security](https://krebsonsecurity.com/2018/07/google-security-keys-neutralized-employee-phishing/)  
<sup id="fn7">[⁷](#f7)</sup> [Defeating Phishing with FIDO2 for ASP.NET - IdentityServer](https://www.identityserver.com/articles/defeating-phishing-with-fido2-for-aspnet)
