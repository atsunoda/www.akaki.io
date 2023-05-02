---
description: A previous article indicated a URL spoofing vulnerability in Slack using link syntax. Subsequently, a URL spoofing vulnerability using a right-to-left override (RTLO or RLO) character was found in many popular messaging apps. This article explains how URL spoofing operates using RTLO. Moreover, it indicates the vulnerability found in a Japanese messaging app and corresponding mitigation measures.
---

# URL Spoofing Using RTLO in Messaging Apps

<p class="modest" align="left">Jan 23, 2023</p>

---

A previous [article](/2020/url_link_spoofing.md) indicated a URL spoofing vulnerability in Slack using link syntax. Subsequently, a URL spoofing vulnerability using a right-to-left override (RTLO or RLO) character was found in many popular messaging apps. This article explains how URL spoofing operates using RTLO. Moreover, it indicates the vulnerability found in a Japanese messaging app and corresponding mitigation measures.

## Summary

URL spoofing using RTLO is a vulnerability that enables visual spoofing by reversing the display order of URL strings. In other words, it can display the URL of an illegitimate website as that of a legitimate one; thus, attackers can direct victims to illegitimate websites. Therefore, it can be exploited for phishing scams and malware distribution. Vulnerability was also found in +Message, a messaging app co-developed by Japanese mobile network operators. The developers mitigated this by displaying a warning dialog before accessing an illegitimate website.

## Spoofing Mechanism

RTLO is a Unicode control character (U+202E) that supports languages using right-to-left text<sup id="f1">[¹](#fn1)</sup>. As in this article, English text is written from left to right. In contrast, some languages, such as Arabic and Hebrew, are written from right to left. Using RTLO within a left-to-right text reverses a string from right to left. That is, the string `\u202EHello, World!` is displayed as `!dlroW ,olleH`.

This characteristic of RTLO has the potential to be exploited for visual spoofing<sup id="f2">[²](#fn2)</sup>. For example, the filename `mal\u202Efdp.exe` is displayed as `malexe.pdf`, thus enabling attackers to spoof file extensions<sup id="f3">[³](#fn3)</sup>. In the real world, attackers direct victims to malware and phishing webpages by attaching files with spoofed extensions to emails and messages<sup id="f4">[⁴](#fn4)</sup> <sup id="f5">[⁵](#fn5)</sup>. Additionally, the spoofability of source code has been demonstrated<sup id="f6">[⁶](#fn6)</sup> <sup id="f7">[⁷](#fn7)</sup>.

## URL Spoofing

Generally, URLs remain left-to-right, even within the right-to-left text. The URL Standard defines that browsers should render URLs containing bidirectional (mixtures of left-to-right and right-to-left) text as left-to-right<sup id="f8">[⁸](#fn8)</sup>. This definition seems to be accepted by many users of right-to-left languages<sup id="f9">[⁹](#fn9)</sup>. For example, the URL in Arabic Wikipedia is written from left to right as follows<sup id="f10">[¹⁰](#fn10)</sup>.

<blockquote dir="rtl" lang="ar">محدد موقع الموارد المُوحّد (بالإنجليزية: Uniform Resource Locator اختصاراً URL)  ويعد جزء من معرف الموارد الموحد وبواسطته يتم تحديد مواقع الانترنت. وهو ذلك العنوان الذي تكتبه في شريط العنوان للذهاب إلى مواقع الإنترنت ويسبقه تحديد البروتوكول مثال //:http أو البروتوكول //:ftp وعلى سبيل المثال عنوان هذه الصفحة هو http://ar.wikipedia.org يضم العنوان بالترتيب:</blockquote>

However, the globally popular messaging apps were rendering URLs containing RTLO as links with reversed strings<sup id="f11">[¹¹](#fn11)</sup> <sup id="f12">[¹²](#fn12)</sup>. That is, the URL string `\u202Ehttps://evil.akaki.io/#moc.elgoog` was displayed as the following link, which links to `evil.akaki.io`, not `google.com`.

<p align="right"><bdo dir="rtl"><a href="https://evil.akaki.io/#moc.elgoog">https://evil.akaki.io/#moc.elgoog</a></bdo></p>

Thus, URLs can be visually spoofed by reversing the display order of the strings. Therefore, attackers can direct victims who are accustomed to a left-to-right language to phishing websites and malware distributors. If messages are displayed on the left side owing to the Bubble UI, the victims will misread the following spoofed URLs.

<p style="background-color: #ebedf0;width: fit-content;padding: 3px 11px;border-start-start-radius: 4px;border-end-start-radius: 18px;border-start-end-radius: 18px;border-end-end-radius: 18px;"><bdo dir="rtl"><a href="https://evil.akaki.io/#moc.elgoog">https://evil.akaki.io/#moc.elgoog</a></bdo></p>

##  Vulnerability in +Message

As with many messaging apps, the URL spoofing vulnerability (CVE-2022-43543) caused by RTLO was found in +Message<sup id="f13">[¹³](#fn13)</sup>. +Message is a messaging app for iOS and Android that provides SMS and RCS, which was co-developed by Japanese mobile network operators<sup id="f14">[¹⁴](#fn14)</sup>. Figure 1 shows the URL spoofing vulnerability in +Message. In the figure, the attacker uses macOS (left) and Android (center), whereas the victim uses iOS (right). The attacker copies the URL string containing the RTLO using `pbcopy` and then pastes it into the Android device via scrcpy. On tapping the link received from the attacker, the victim is directed to `evil.akaki.io`, not `google.com`.

<video controls poster="/assets/2023/url_spoofing_using_rtlo_in_messaging_apps/27_figure1.png" src="/assets/2023/url_spoofing_using_rtlo_in_messaging_apps/27_figure1.mp4" type="video/mp4"></video>
<p class="modest" align="center">Figure 1: Demonstration of URL spoofing in +Message.</p>

## Mitigation Measures

Developers should consider implementing some mitigation measures against URL spoofing using RTLO. The URL Standard advises to only render a URL's host when a URL contains bidirectional text<sup id="f8">[⁸](#fn8)</sup>. Furthermore, the specifications of the Internationalized Resource Identifier (IRI), which extends the character set of URLs, specify that IRIs must not contain bidirectional formatting characters when they are displayed<sup id="f15">[¹⁵](#fn15)</sup>. Therefore, a process is required to avoid visual spoofing before rendering the URL string as a link.

Alternatively, a process to verify a URL before accessing a resource can mitigate risk. The developers of +Message implemented a mitigation measure that warns of risk through a dialog before accessing a spoofed URL. Figure 2 illustrates the warning dialog that appears in the mitigated app. The URL in the message is reversed, whereas the URL in the dialog is not. Therefore, users can detect spoofing before they are directed to illegitimate websites.

<p align="center"><video controls poster="/assets/2023/url_spoofing_using_rtlo_in_messaging_apps/27_figure2.png" src="/assets/2023/url_spoofing_using_rtlo_in_messaging_apps/27_figure2.mp4" type="video/mp4" width="300px"></video></p>
<p class="modest" align="center">Figure 2: Warning dialog against a spoofed URL.</p>

## Conclusion and Future Work

URLs containing RTLO may be rendered as links that are misread by users accustomed to left-to-right language. Thus, developers should avoid rendering such links or warn users about them. Owing to previous research, globally popular messaging apps have already mitigated the risk of URL spoofing; however, apps only used in some countries may still be vulnerable. Therefore, further research is required to focus on such apps. Not limited to URL spoofing, RTLO has the potential to be exploited for social engineering using spoofed usernames and bypassing posting restrictions using spoofed words and sentences. We should recognize the existence of such spoofabilities.

---

<sup id="fn1">[¹](#f1)</sup> [UNICODE BIDIRECTIONAL ALGORITHM - Unicode Standard Annex #9](https://www.unicode.org/reports/tr9/tr9-46.html)  
<sup id="fn2">[²](#f2)</sup> [UNICODE SECURITY CONSIDERATIONS - Unicode Technical Report #36](https://www.unicode.org/reports/tr36/)  
<sup id="fn3">[³](#f3)</sup> [Masquerading: Right-to-Left Override, Sub-technique T1036.002 - MITRE ATT&CK](https://attack.mitre.org/techniques/T1036/002/)  
<sup id="fn4">[⁴](#f4)</sup> [Zero-day vulnerability in Telegram - SECURELIST by Kaspersky](https://securelist.com/zero-day-vulnerability-in-telegram/83800/)  
<sup id="fn5">[⁵](#f5)</sup> [How Hackers Are Using a 20-Year-Old Text Trick to Phish Microsoft 365 Users - Vade](https://www.vadesecure.com/en/blog/how-hackers-are-using-a-20-year-old-text-trick-to-phish-microsoft-365-users)  
<sup id="fn6">[⁶](#f6)</sup> [Ethereum Smart Contracts Exploitation Using Right-To-Left-Override Character - Skylight Cyber](https://skylightcyber.com/2019/05/12/ethereum-smart-contracts-exploitation-using-right-to-left-override-character/)  
<sup id="fn7">[⁷](#f7)</sup> [Trojan Source: Invisible Vulnerabilities - arXiv](https://arxiv.org/abs/2111.00169)  
<sup id="fn8">[⁸](#f8)</sup> [4.8.3. Internationalization and special characters - URL Living Standard](https://url.spec.whatwg.org/#url-rendering-i18n)  
<sup id="fn9">[⁹](#f9)</sup> [1.4.1 Overall Presentation in a Bidirectional Language - IRIStatus - W3C](https://www.w3.org/International/wiki/IRIStatus#Overall_Presentation_in_a_Bidirectional_Language)  
<sup id="fn10">[¹⁰](#f10)</sup> <a href="https://ar.wikipedia.org/wiki/%D9%85%D8%AD%D8%AF%D8%AF_%D9%85%D9%88%D9%82%D8%B9_%D8%A7%D9%84%D9%85%D9%88%D8%A7%D8%B1%D8%AF_%D8%A7%D9%84%D9%85%D9%88%D8%AD%D8%AF" style="float: right;">محدد موقع الموارد الموحد - ويكيبيديا</a>  
<sup id="fn11">[¹¹](#f11)</sup> [Exploit: RTLO Injection URI Spoofing: WhatsApp, iMessage (Messages app), Instagram, Facebook Messenger - Sick.Codes](https://sick.codes/sick-2022-40/)  
<sup id="fn12">[¹²](#f12)</sup> [Signal client for iOS version 5.33.2 and below are vulnerable to RTLO Injection URI Spoofing - Sick.Codes](https://sick.codes/sick-2022-42/)  
<sup id="fn13">[¹³](#f13)</sup> [JVN#43561812: +Message App improper handling of Unicode control characters - JVN](https://jvn.jp/en/jp/JVN43561812/index.html)  
<sup id="fn14">[¹⁴](#f14)</sup> [RCS Business Messaging in Japan - GSMA](https://www.gsma.com/futurenetworks/wp-content/uploads/2019/12/2_RCS-Business-Messaging-in-Japan-English-spreads-combined-low-res.pdf)  
<sup id="fn15">[¹⁵](#f15)</sup> [4.1. Logical Storage and Visual Presentation - RFC 3987: Internationalized Resource Identifiers (IRIs)](https://www.rfc-editor.org/rfc/rfc3987.html#section-4.1)
