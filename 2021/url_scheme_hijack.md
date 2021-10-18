# カスタムURLスキームの乗っ取りとその対策

<p class="modest" align="left">May 17, 2021</p>

---

カスタムURLスキームは、モバイルアプリ内のコンテンツへ直接誘導するディープリンクに広く利用されている<sup id="f1">[¹](#fn1)</sup>。そのような中で、2020年3月にLINEはカスタムURLスキーム `line://` の使用を非推奨とした<sup id="f2">[²](#fn2)</sup>。非推奨の理由をLINEは「乗っ取り攻撃が可能なため」と説明し、代わりにHTTP URLスキームによるリンクを推奨している。この変更に対して私は、なぜHTTP URLスキームによるリンクだと乗っ取り攻撃を防げるのか疑問を抱いた。この疑問に答えるためにLINEアプリの乗っ取りを試み、対策の有効性を確認した。

## 要約

HTTP URLスキームによるディープリンクは対象のアプリを一意に特定できるため、不正アプリによるリンクの乗っ取りが発生しない。カスタムURLスキームでは複数のアプリが同じスキームを宣言できるため、モバイルOSはリンクの対象とするアプリを一意に特定できない。それにより、正規アプリのスキームを偽装した不正アプリによってリンクを乗っ取られる恐れがある。しかし、HTTP URLスキームではWebサイトに紐付くアプリをリンクの対象とするため、不正アプリによる偽装が困難となり乗っ取り攻撃を防止できる。

## カスタムURLスキームの抱える問題

アプリ開発者は任意の文字列をカスタムURLスキームとして宣言できる。URLの先頭であるスキーム部で対象のアプリを示すカスタムURLスキームは、モバイルアプリ内の特定の画面や機能に直接誘導するためのリンク（以下、ディープリンク）に活用されている。例えば、AndroidとiOSのLINEアプリ 11.7.0は [`line://nv/chat`](line://nv/chat) というディープリンクによりトーク画面を開く。このようにLINEアプリが独自のスキームによるリンクを実現できるのは、AndroidとiOSが任意のスキームによるディープリンクの機能を提供しているからである<sup id="f3">[³](#fn3)</sup> <sup id="f4">[⁴](#fn4)</sup>。

ただし、同じカスタムURLスキームを宣言した複数のアプリをOSは一意に特定できない。複数のアプリが同じスキームを宣言していた場合の挙動はOSごとに異なり、Androidでは起動するアプリの選択ダイアログを表示する<sup id="f5">[⁵](#fn5)</sup>。iOSでは具体的な挙動は示されていないが、過去のiOS公式プログラミングガイドでは「対象のアプリを決定するプロセスはない」と述べている<sup id="f6">[⁶](#fn6)</sup>。また、iOSアプリには同じスキームを宣言した複数のアプリを区別するための識別子を指定できるが、この識別子は「他のアプリへのリンクを防ぐものではない」と開発者ドキュメントで説明している<sup id="f4">[⁴](#fn4)</sup>。どちらのOSもカスタムURLスキームだけでは対象のアプリを一意に特定できない仕様である。

## カスタムURLスキームの偽装による悪用

正規アプリと同じカスタムURLスキームを宣言した不正アプリをインストールすると、正規アプリへのリンクを乗っ取られる恐れがある。正規アプリのカスタムURLスキームを偽装した国内の悪用事例として、2015年にフリマアプリがLINEを含む既存アプリのスキームを大量に宣言し、正規アプリへのリンクを乗っ取って被害者を広告画面に誘導していた<sup id="f7">[⁷](#fn7)</sup>。中国ではライブ配信アプリがAlipayのスキームを乗っ取り、支払い手続きを妨害していた事例もある<sup id="f8">[⁸](#fn8)</sup>。さらに、WeChatやAlipayのスキームを乗っ取って偽の認証画面に誘導し、入力された認証情報を窃取する概念実証も示されている<sup id="f9">[⁹](#fn9)</sup>。これらは古いiOSだけで確認された事象であるため、Androidも含めた最新のOSでスキームの偽装による乗っ取りの可能性を検証した。

### Androidでのディープリンクの乗っ取り

Androidでは起動するアプリとして不正アプリが選択された場合に乗っ取り攻撃が可能となる。AndroidアプリでカスタムURLスキームを宣言するにはマニフェストにインテントフィルタとして指定する<sup id="f3">[³](#fn3)</sup>。Android版LINEアプリのAndroidManifest.xmlの `android:scheme` には以下のようにスキームが指定されている。

```xml
<activity android:theme="@ref/0x7f140179" android:name="jp.naver.line.android.activity.schemeservice.LineSchemeServiceActivity" android:exported="true" android:launchMode="3">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="line" />
    </intent-filter>
```

乗っ取りの可否を検証するため、LINEアプリと同じカスタムURLスキームを以下のように宣言した偽アプリを実装し、リリースビルドしたAPKをWebサイト経由でAndroid 11の端末にインストールする。

```xml
<activity android:name=".MainActivity">
    ...
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="line" />
    </intent-filter>
```

なお、インストールする際に不明なアプリのインストールを許可し、Google Play プロテクトの警告を無視する必要がある。LINEアプリがインストール済みの端末では、偽アプリのインストール後に `line://` スキームのリンクをタップすると選択ダイアログが表示される。ダイアログ上の偽アプリを選択すると偽アプリが起動する（図1）。不正アプリが表示名やアイコンも偽装していた場合、正規アプリと誤認した被害者が不正アプリを選択することが想定される。

<p align="center"><img src="/assets/2021/url_scheme_hijack/21_figure1.gif" width="300" alt="figure1"></p>
<p class="modest" align="center">図1. Android版LINEアプリへのリンクを偽アプリが乗っ取る</p>

### iOSでのディープリンクの乗っ取り

iOSでは正規アプリよりも後に不正アプリがインストールされた場合に乗っ取り攻撃が可能となる。iOSアプリでカスタムURLスキームを宣言するには、Xcodeのプロジェクトエディターから \[Info\] &gt; \[URL Types\] &gt; \[URL Schemes\] に指定する<sup id="f4">[⁴](#fn4)</sup>。iOS版LINEアプリのInfo.plistの `CFBundleURLSchemes` には以下のようにスキームが指定されている。

```xml
<key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleTypeRole</key>
            <string>Editor</string>
            <key>CFBundleURLIconFile</key>
            <string>Icon</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>line</string>
            </array>
            <key>CFBundleURLName</key>
            <string>jp.naver.line</string>
        </dict>
```

iOSの偽アプリではXcodeからLINEアプリと同じカスタムURLスキームを宣言する（図2）。

<p align="center"><img src="/assets/2021/url_scheme_hijack/21_figure2.png" alt="figure2"></p>
<p class="modest" align="center">図2. LINEアプリと同じカスタムURLスキームを偽アプリに指定する</p>

端末上でのLINEアプリとの重複を避けるため、偽アプリのBundle IDを `jp.naver.line.fake` と設定する。その後、Ad Hoc配信で偽アプリのIPAを生成し、Webサイト経由でiOS 14.5.1の端末にインストールする。なお、インストールする端末のDevice ID（UDID）を事前にProvisioning profileに登録する必要がある。偽アプリのインストール後に `line://` スキームのリンクをタップすると確認ダイアログが表示される。ダイアログ上でアプリの起動に同意すると偽アプリが起動する（図3）。偽アプリの表示名を `LINE` と設定することでダイアログ上のメッセージも偽装できる。

<p align="center"><img src="/assets/2021/url_scheme_hijack/21_figure3.gif" width="300" alt="figure3"></p>
<p class="modest" align="center">図3. iOS版LINEアプリへのリンクを偽アプリが乗っ取る</p>

iOSの不正アプリをAd Hocで配信するには、攻撃者は事前に被害者の端末のUDIDを入手する必要がある。特定の被害者を標的とする攻撃者は、端末への物理アクセスや被害者へのソーシャルエンジニアリングによりUDIDを入手する。無差別な攻撃には過去に漏洩したUDIDのリストを使用するなどが想定される。なお、UDIDを必要としないEnterprise配信はより有効な攻撃ベクトルになり得るが<sup id="f10">[¹⁰](#fn10)</sup>、個人ではApple Developer Enterprise Programに登録できないため検証していない。

## HTTP URLスキームによる解決

不正アプリによるディープリンクの乗っ取りを防ぐには、HTTP URLスキームによりWebサイトとの紐付きでアプリを特定する。LINEアプリのトーク画面を開くディープリンクには [`https://line.me/R/nv/chat`](https://line.me/R/nv/chat) というURLが推奨されている。このようなHTTP URLスキームによるディープリンクの機能を、Androidでは「Android アプリリンク」<sup id="f11">[¹¹](#fn11)</sup>、iOSでは「ユニバーサルリンク」<sup id="f12">[¹²](#fn12)</sup>という名称で提供している。どちらも基本的な仕組みは同じで、アプリが宣言するドメインのWebサイトに配置されたJSONファイルをOSは参照し、そのファイルに記載されているアプリをリンク対象と見なす。WebサイトのドメインはDNSによって一意性が保証されるため、ドメインやWebサーバーを乗っ取られない限り不正アプリによる偽装を防止できる。

ただし、HTTP URLスキームによるディープリンクの設定を誤ると、OSはアプリとドメインの紐付きを適切に検証できない。ディープリンクの安全性を調査したLiuらの研究によると、2017年の時点でAndroid アプリリンクを実装しているアプリの95.3％はリンクの検証を有効にする設定を忘れていた<sup id="f13">[¹³](#fn13)</sup>。他にも設定値の記述が誤っているケースや、HTTPSではなくHTTPで参照ファイルを配置しているケースなどもあり、適切に検証できているアプリは全体の2％だった。また、iOSのユニバーサルリンクでも全体の14％のWebサイトが誤ってHTTPで参照ファイルを配置していた。このような観点も含め、LINEアプリでのHTTP URLスキームによるディープリンクの有効性を確認した。

### Android アプリリンクによる乗っ取りの防止

Android アプリリンクは署名証明書のフィンガープリントをもとに対象のアプリを特定する。Android アプリリンクを宣言するには、インテントフィルタの `android:scheme` にHTTP URLスキームを指定し、`android:host` に参照先のWebサイトのドメインを指定する<sup id="f14">[¹⁴](#fn14)</sup>。さらに、インテントフィルタにリンクの検証を有効にするための `android:autoVerify="true"` を設定する<sup id="f15">[¹⁵](#fn15)</sup>。Android版LINEアプリのAndroidManifest.xmlには、Android アプリリンクの一部が以下のように宣言されている。参照先のWebサイトのドメインのひとつに `line.me` が指定されており、`android:autoVerify="true"` はアクティビティのエイリアスに設定されている。

```xml
<activity android:theme="@ref/0x7f140179" android:name="jp.naver.line.android.activity.schemeservice.LineSchemeServiceActivity" android:exported="true" android:launchMode="3">
    ...
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" android:host="line.naver.jp" />
        <data android:scheme="http" android:host="line.me" />
        <data android:scheme="https" android:host="line.naver.jp" />
        <data android:scheme="https" android:host="line.me" />
        <data android:pathPrefix="/R/" />
    </intent-filter>
    ...
<activity-alias android:label="@ref/0x7f130472" android:name="jp.naver.line.android.IdentityRequiredSchemeServiceActivity" android:exported="true" android:targetActivity="jp.naver.line.android.activity.schemeservice.LineSchemeServiceActivity">
    <intent-filter android:autoVerify="true">
```

Androidが参照するWebサイトには、リンク対象のアプリを記載したDigital Asset Linksファイルを `/.well-known/assetlinks.json` として配置する。ファイルには対象とするアプリのパッケージ名（アプリケーション ID）と、アプリを署名した証明書のSHA256フィンガープリントを記載する。LINEアプリが参照するWebサイトである `line.me` には以下のファイルが配置されている。

[https://line.me/.well-known/assetlinks.json](https://line.me/.well-known/assetlinks.json)

```js
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "jp.naver.line.android",
    "sha256_cert_fingerprints":
    ["E6:82:FE:0B:CD:60:90:7D:FE:D5:15:E0:B8:A4:DE:03:AA:1C:28:1D:11:1A:07:83:39:86:60:2B:60:98:AF:D2"]
  }
}]
```

Digital Asset Linksファイルに記載されたフィンガープリントを不正アプリは偽装できないため、Android アプリリンクは乗っ取り攻撃を防止できる。Androidアプリを端末にインストールするにはアプリ開発者の証明書でAPKを署名する必要がある。LINEアプリを署名した証明書のフィンガープリントは以下であり、Digital Asset Linksファイルのフィンガープリントと一致する。

```console
$ keytool -printcert -file META-INF/BNDLTOOL.RSA | grep "SHA256: " | cut -d " " -f 3
E6:82:FE:0B:CD:60:90:7D:FE:D5:15:E0:B8:A4:DE:03:AA:1C:28:1D:11:1A:07:83:39:86:60:2B:60:98:AF:D2
```

この証明書の秘密鍵はLINE社が所有しているため、攻撃者はLINEアプリと同じ証明書で不正アプリを署名できない。偽アプリを署名しているのは私が所有する証明書であり、フィンガープリントは以下である。Digital Asset Linksファイルに記載されたフィンガープリントとは一致しない。

```console
$ keytool -list -v -keystore my-release-key.jks | grep "SHA256: " | cut -d " " -f 3
5D:B3:E4:67:EF:A7:C4:CF:1D:8A:4A:9B:3D:1A:AF:F7:EA:4A:9D:CB:5D:D8:8A:23:CE:A8:65:10:CC:5E:FF:A0
```

そのため、偽アプリのマニフェストで以下のようにLINEアプリと同じHTTP URLスキームを宣言しても、Android アプリリンクは選択を求めることなくLINEアプリを起動する（図4）。不正アプリがインストールされた状態でもAndroid アプリリンクの乗っ取りは発生しない。

```xml
<activity android:name=".MainActivity">
    ...
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="line.me" />
        <data android:pathPrefix="/R/" />
    </intent-filter>
```

<p align="center"><img src="/assets/2021/url_scheme_hijack/21_figure4.gif" width="300" alt="figure4"></p>
<p class="modest" align="center">図4. Android アプリリンクは偽アプリに乗っ取られない</p>

### ユニバーサルリンクによる乗っ取りの防止

ユニバーサルリンクはiOSアプリの識別子であるApp IDをもとに対象のアプリを特定する。iOSアプリでユニバーサルリンクを宣言するには、Xcodeのプロジェクトエディターから \[Signing & Capabilities\] に \[Associated Domains\] 項目を追加し、そこにアプリと紐付けるドメインを `applinks:` というプレフィックス付きで指定する<sup id="f16">[¹⁶](#fn16)</sup>。iOS版LINEアプリのIPAに含まれるEntitlementsには以下のようにドメインが指定されている。

```console
$ codesign -d --entitlements :- Payload/LINE.app | grep -A 6 "associated-domains"
<key>com.apple.developer.associated-domains</key>
<array>
    <string>webcredentials:line.me</string>
    <string>applinks:line.me</string>
    <string>applinks:access-auto.line.me</string>
    <string>applinks:liff.line.me</string>
</array>
```

iOSが参照するWebサイトには、リンク対象のアプリを記載したapple-app-site-associationファイルを `.well-known` ディレクトリに配置する。ファイルの `applinks` 項目に対象とするアプリのApp IDを記載する。LINEアプリが参照するWebサイトである `line.me` に配置されたファイルには以下のように記載されている。

[https://line.me/.well-known/apple-app-site-association](https://line.me/.well-known/apple-app-site-association)

```js
"applinks": {
    "apps": [],
    "details": [
        {
            "appID": "ZW4U99SQQ3.jp.naver.line",
            "paths": [
                "/R/*",
                "/S/*",
                "/D"
            ]
        },
```

apple-app-site-associationファイルに記載されたApp IDを不正アプリは偽装できないため、ユニバーサルリンクは乗っ取り攻撃を防止できる。iOSアプリの識別子であるApp IDは、Appleによって払い出されるApp ID Prefixとアプリ開発者が設定するBundle IDを連結した値である。LINEアプリのEntitlementsに指定されているApp IDは以下であり、apple-app-site-associationファイルのApp IDと一致する。このApp IDの前方 `ZW4U99SQQ3` がLINE社の開発者アカウントに払い出されたApp ID Prefixである。

```console
$ codesign -d --entitlements :- Payload/LINE.app | grep -A 1 "application-identifier"
<key>application-identifier</key>
<string>ZW4U99SQQ3.jp.naver.line</string>
```

私の開発者アカウントに払い出されたApp ID Prefixは `363PDYH48N` であるため、偽アプリのApp IDは `363PDYH48N.jp.naver.line.fake` となる。apple-app-site-associationファイルに記載されたApp IDとは一致しない。そのため、偽アプリにLINEアプリと同じドメインを指定しても（図5）、ユニバーサルリンクはLINEアプリを起動する（図6）。不正アプリがインストールされた状態でもユニバーサルリンクの乗っ取りは発生しない。

<p align="center"><img src="/assets/2021/url_scheme_hijack/21_figure5.png" alt="figure5"></p>
<p class="modest" align="center">図5. LINEアプリと同じドメインを偽アプリに指定する</p>

<p align="center"><img src="/assets/2021/url_scheme_hijack/21_figure6.gif" width="300" alt="figure6"></p>
<p class="modest" align="center">図6. ユニバーサルリンクは偽アプリに乗っ取られない</p>

## 所感

カスタムURLスキームの偽装による乗っ取り攻撃は、被害者の端末に不正アプリがインストールされていることが前提となる。この前提により悪用は難しいと考えていたが、昨今の宅配便業者をかたるスミッシングの被害状況から察するに、不正アプリをインストールするよう被害者を誘導することは現実的に可能である<sup id="f17">[¹⁷](#fn17)</sup>。また、不正アプリがGoogle PlayやApp Storeの審査をすり抜け、信頼されたアプリとして配信されていた事例もある。HTTP URLスキームによるディープリンクをiOSは強く推奨しており<sup id="f4">[⁴](#fn4)</sup>、AndroidもカスタムURLスキームより安全と説明している<sup id="f11">[¹¹](#fn11)</sup>。そのため、今後はLINEのようなセキュリティへの配慮が求められるアプリからカスタムURLスキームの廃止が進んでいくと予想する。

---

<sup id="fn1">[¹](#f1)</sup> [Adoption of URL Schemes, Universal Links and App Indexing by Leading Online Retailers - Pure Oxygen Labs](https://pureoxygenlabs.com/mobile-deep-linking/adoption-of-universal-links-and-app-indexing-leading-online-retailers/)  
<sup id="fn2">[²](#f2)</sup> [LINE URLスキームの「line://」は非推奨になりました - LINE Developers](https://developers.line.biz/ja/news/2020/03/25/line-url-scheme-deprecation/)  
<sup id="fn3">[³](#f3)</sup> [アプリ コンテンツ用のディープリンクを作成する - Android Developers](https://developer.android.com/training/app-links/deep-linking)  
<sup id="fn4">[⁴](#f4)</sup> [Defining a Custom URL Scheme for Your App - Apple Developer Documentation](https://developer.apple.com/documentation/xcode/defining-a-custom-url-scheme-for-your-app)  
<sup id="fn5">[⁵](#f5)</sup> [別のアプリにユーザーを送信する - Android Developers](https://developer.android.com/training/basics/intents/sending#disambiguation-dialog)  
<sup id="fn6">[⁶](#f6)</sup> [Inter-App Communication - Apple Developer Documentation](https://web.archive.org/web/20190514160038/https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html)  
<sup id="fn7">[⁷](#f7)</sup> [\(前半\)URLスキームを扱う上で気をつけたいあるアプリ - もう一人のY君](http://blog.thetheorier.com/entry/2015/05/03/143309)  
<sup id="fn8">[⁸](#f8)</sup> [URL Masques on App Store - FireEye](https://www.fireeye.com/blog/threat-research/2015/04/url_masques_on_apps.html)  
<sup id="fn9">[⁹](#f9)</sup> [iOS URL Scheme 劫持-在未越狱的 iPhone 6上盗取支付宝和微信支付的帐号密码 - WooYun知识库](https://web.archive.org/web/20160707133615/http://drops.wooyun.org:80/papers/5309)  
<sup id="fn10">[¹⁰](#f10)</sup> [VB2014 paper: Apple without a shell – iOS under targeted attack - Virus Bulletin](https://www.virusbulletin.com/virusbulletin/2014/11/paper-apple-without-shell-ios-under-targeted-attack)  
<sup id="fn11">[¹¹](#f11)</sup> [Android アプリリンクの処理 - Android Developers](https://developer.android.com/training/app-links)  
<sup id="fn12">[¹²](#f12)</sup> [Allowing Apps and Websites to Link to Your Content - Apple Developer Documentation](https://developer.apple.com/documentation/xcode/allowing-apps-and-websites-to-link-to-your-content)  
<sup id="fn13">[¹³](#f13)</sup> [Measuring the Insecurity of Mobile Deep Links of Android - USENIX](https://www.usenix.org/conference/usenixsecurity17/technical-sessions/presentation/liu)  
<sup id="fn14">[¹⁴](#f14)</sup> [Android アプリリンクを追加する - Android Developers](https://developer.android.com/studio/write/app-link-indexing)  
<sup id="fn15">[¹⁵](#f15)</sup> [Android アプリリンクを検証する - Android Developers](https://developer.android.com/training/app-links/verify-site-associations)  
<sup id="fn16">[¹⁶](#f16)</sup> [Supporting Associated Domains - Apple Developer Documentation](https://developer.apple.com/documentation/xcode/supporting-associated-domains)  
<sup id="fn17">[¹⁷](#f17)</sup> [宅配便業者をかたる偽ショートメッセージに引き続き注意！ - IPA 情報処理推進機構](https://www.ipa.go.jp/security/anshin/mgdayori20200220.html)
