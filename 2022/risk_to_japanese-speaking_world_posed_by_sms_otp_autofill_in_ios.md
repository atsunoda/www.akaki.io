# Risk to Japanese-speaking World posed by SMS OTP Autofill in iOS

<p class="modest" align="left">Nov 14, 2022</p>

---

iOS, which is a mobile operating system developed by Apple, includes a feature that allows users to autofill website forms with a one-time password (OTP) sent during short message service (SMS) authentication. This feature carries the risk of automatically entering OTPs, sent by legitimate services, to phishing websites, a risk that has already been proven for English-language web services. As indicated in my previous [study](/2021/sms_otp_autofill.md), the risk may also extend to Japanese-language web services. In this study, the risk to major web services that send OTPs with Japanese-language SMS messages was studied. Consequently, the risk was demonstrated in 7 of the 10 websites studied. Furthermore, a mitigation measure was proposed for service providers.

## Risk posed by SMS OTP autofill

iOS has a feature that autofills OTPs sent via SMS into website forms<sup id="f1">[¹](#fn1)</sup>. If an OTP is included in the SMS message received by iOS, this feature extracts the value and displays it as an input suggestion at the top of the keyboard. According to Apple’s explanation, the autofill feature in iOS heuristically extracts OTPs from near words like `code` or `passcode` in SMS messages<sup id="f2">[²](#fn2)</sup>. For example, in the SMS message `Your code is 123456.` shown in Figure 1, the numeric value `123456`, which is indicated by the term `code` is extracted as an OTP and displayed as an input suggestion. This feature reduces the time and effort required by iOS users to read the OTP from the received SMS message and type it into the form during SMS authentication.

<p align="center"><img src="/assets/2022/risk_to_japanese-speaking_world_posed_by_sms_otp_autofill_in_ios/26_figure1.png" width="300px" /></p>
<p class="modest" align="center">Figure 1: SMS OTP autofill in iOS.</p>

However, the autofill feature has proven to be a potential success factor in phishing attacks. Gutmann et al. (2019) raised the risk of facilitating misinput to phishing sites because iOS autofills SMS OTPs to non-legitimate sites<sup id="f3">[³](#fn3)</sup>. Furthermore, their study demonstrated the risk that SMS OTPs sent during two-factor authentication at login in English-language web services can be automatically entered into phishing sites. This risk means that in the man-in-the-middle phishing procedure shown in Figure 2, the process of entering the SMS OTP received by the victim from the legitimate site in Step 4 to the phishing site in Step 5 is automatic.

<p align="center"><img src="/assets/2022/risk_to_japanese-speaking_world_posed_by_sms_otp_autofill_in_ios/26_figure2.png" /></p>
<p class="modest" align="center">Figure 2: Procedure to bypass SMS authentication by man-in-the-middle phishing.</p>

If the autofill feature in iOS also extracts OTPs from non-English SMS messages, web services in that language would also be at risk. In my previous study, I demonstrated that the autofill feature in iOS also extracts OTPs from a message involving a Japanese word meaning `code`. Precisely, the autofill feature in iOS extracts OTPs from words containing `認証番号` or `コード` in Japanese SMS messages. Consequently, I hypothesized that the same risk extends to Japanese-language web services that use these words to represent the SMS OTP. In this study, I verified this hypothesis by using web services that provide SMS authentication via Japanese SMS messages.

## Verification of risk in Japanese-speaking world

The risk posed by the autofill feature in iOS was verified by targeting web services that provide SMS authentication for Japanese-language users. The target websites were selected from web services that are widely used in Japan. Specifically, from the Alexa Top Sites in Japan as of August 2021, 10 websites were selected that provide SMS authentication via Japanese SMS messages through two-factor authentication for logging into an account. I tested whether the OTP sent during SMS authentication for the selected web service was automatically entered into the form on the verification website. The [website](https://atsunoda.github.io/demo/sms_otp_autofill/one-time-code.html) implemented in my previous study was used as the verification website and tested using Safari and Chrome on iOS 15.

As a result of the verification, the autofill feature was triggered in SMS messages sent from 7 of the 10 websites. Table 1 shows the SMS messages and autofill status for each web service. The seven websites where autofill was triggered had OTPs represented by words containing `認証番号` or `コード` identified in my previous study. Conversely, out of the three websites where autofill was not triggered, Facebook had no word in the message representing an OTP, and Amazon used the word `ワンタイムパスワード` to represent an OTP. Although Adobe used the word containing `コード` to represent the OTP, autofill was not triggered. This implies that the autofill feature in iOS extracts the OTP by analyzing the context of the SMS message. All results were similar on Safari and Chrome.

<p class="modest" align="center">Table 1: SMS messages and autofill status for each target web service.</p>

| Web services | SMS messages | Autofill status |
| :--- | :--- | :---: |
| Google.com | G-255328 があなたの Google 確認コードです。 | Triggered |
| Yahoo.co.jp | 認証番号: 4290<br />上記の番号を画面へ入力してください。<br />Yahoo! JAPAN | Triggered |
| Amazon.co.jp | 300015は、Amazonのワンタイムパスワードです。誰とも共有しないでください。 | Not |
| Facebook.com | Facebookの二段階認証で683437を使用してください。 | Not |
| Zoom.us | Zoomコード：558324。Zoomに電話番号を確認させるために、このコードを入力してください。他の人とこのコードを共有しないでください。 | Triggered |
| Twitter.com | Twitterの認証コードは753232です。このメッセージには返信できません。 | Triggered |
| Instagram.com | あなたの非公開Instagramコードは407 072です。コードは他の人にシェアしないでください。 | Triggered |
| Microsoft.com | Microsoft アカウントのセキュリティ コードとして 2158992 を使います | Triggered |
| Apple.com | Apple IDコードは次の通りです：984634。コードを共有しないでください。 | Triggered |
| Adobe.com | 596044 はお客様の Adobe コードです | Not |

The results indicate that the risk raised by Gutmann et al. (2019) extends to the Japanese-speaking world. Their study demonstrated the risk of SMS OTPs sent from English-language web services being automatically entered into phishing sites by iOS. Similarly, I herein demonstrated that SMS OTPs sent from Japanese-language web services are automatically entered into non-legitimate sites. Figure 3 shows an OTP in an Apple SMS message that is automatically entered into the verification website. Thus, the risk posed by the autofill feature in iOS was demonstrated for Japanese-language web services.

<p align="center"><img src="/assets/2022/risk_to_japanese-speaking_world_posed_by_sms_otp_autofill_in_ios/26_figure3.png" width="300px" /></p>
<p class="modest" align="center">Figure 3: Autofill with an Apple SMS message.</p>

## Mitigation Measure

As a mitigation measure against the risk posed by SMS OTP autofill, a specification is proposed for the SMS sender to indicate to the mobile OS as to on which websites should the OTP be entered<sup id="f4">[⁴](#fn4)</sup>. This specification defines an SMS message format that binds the OTP to a website. Specifically, binding these is possible by specifying the domain of the website after the `@` and the value of the OTP after the `#` in the last line of the SMS message. Based on this format, the mobile OS that receives the SMS message mechanically determines the OTP and website to enter it. For this mechanism to work, both the mobile OS provider and SMS sender, the web service provider, must comply with the specification. As the autofill feature based on the specification has been implemented since iOS 14<sup id="f5">[⁵](#fn5)</sup>, I suggested the implementation of the specification to seven web service providers with demonstrated risk. As of November 2022, the specification was implemented in Apple SMS messages, and, as shown in Figure 4, the autofill feature is no longer triggered on websites that differ from the specified domain.

<p align="center"><img src="/assets/2022/risk_to_japanese-speaking_world_posed_by_sms_otp_autofill_in_ios/26_figure4.png" width="300px" /></p>
<p class="modest" align="center">Figure 4: Apple SMS message with the mitigation measure.</p>

## Conclusion and Future Work

The risk posed by SMS OTP autofill in iOS to Japanese-language web services was demonstrated. In this study, I proved the risk of automatically entering OTPs into forms on phishing websites using iOS for web services that send OTPs with Japanese-language SMS messages. While this study focused on the risk to web services, the risk posed by the autofill feature also extends to smartphone apps<sup id="f6">[⁶](#fn6)</sup>. Future research should demonstrate the risk to smartphone apps using Japanese-language SMS messages. Furthermore, research is required in other languages such as Chinese, Spanish, and Hindi.

---

<sup id="fn1">[¹](#f1)</sup> [Automatically fill in SMS passcodes on iPhone - Apple Support](https://support.apple.com/guide/iphone/automatically-fill-in-sms-passcodes-iphc89a3a3af/ios)  
<sup id="fn2">[²](#f2)</sup> [Automatic Strong Passwords and Security Code AutoFill - Apple Developer](https://developer.apple.com/videos/play/wwdc2018/204/?time=1572)  
<sup id="fn3">[³](#f3)</sup> [Taken Out of Context: Security Risks with Security Code AutoFill in iOS & macOS - WAY 2019](https://wayworkshop.org/2019/papers/way2019-gutmann.pdf)  
<sup id="fn4">[⁴](#f4)</sup> [Origin-bound one-time codes delivered via SMS - WICG](https://wicg.github.io/sms-one-time-codes/)  
<sup id="fn5">[⁵](#f5)</sup> [Enhance SMS-delivered code security with domain-bound codes - Apple Developer](https://developer.apple.com/news/?id=z0i801mg)  
<sup id="fn6">[⁶](#f6)</sup> [On the Insecurity of SMS One-Time Password Messages against Local Attackers in Modern Mobile Devices - NDSS 2021](https://www.ndss-symposium.org/wp-content/uploads/ndss2021_3B-4_24212_paper.pdf)
