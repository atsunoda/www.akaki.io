---
description: 'May 27, 2019'
---

# オリジンIPの特定によるクラウド型WAFのバイパス

昨年末に「[How i was able to pwned application by Bypassing Cloudflare WAF](https://medium.com/bugbountywriteup/bypass-cloudflare-waf-to-pwned-application-2c9e4f862319)」を読んで、CloudflareのWAFをバイパスする方法とそれがバグバウンティで認定された事例を知った。記事を書いた[@vis\_hacker](https://twitter.com/vis_hacker)氏は調査に「[CloudFlair](https://github.com/christophetd/CloudFlair)」というツールを使用しており、このツールを開発した[@christophetd](https://twitter.com/christophetd)氏も同様の方法で報奨金を獲得していた¹。

Cloudflareに限らずクラウド型WAFのバイパスは2016年頃には既に話題になっており、論文も書かれていた²。2013年のBlackHat USAではDDoS保護のバイパスとして発表され³、DDoS保護サービスを提供するベンダーが注意喚起を行なっている⁴ ⁵。脆弱性として興味深かったので詳細をまとめておく。

## クラウド型WAFの仕組み

ハードウェアを設置するようなアプライアンス型のWAFとは異なり、クラウド型（SaaS型）のWAFはDNSの切り替えだけで導入できる。保護するWebサイトのドメインをWAFセンターのIPアドレスに名前解決させることで、WAFセンターはリバースプロキシとしてクライアント／サーバー間の通信に介入して攻撃を遮断する。

![&#x56F3;1. &#x30AF;&#x30E9;&#x30A6;&#x30C9;&#x578B;WAF&#x306F;&#x30EA;&#x30D0;&#x30FC;&#x30B9;&#x30D7;&#x30ED;&#x30AD;&#x30B7;&#x3068;&#x3057;&#x3066;&#x901A;&#x4FE1;&#x306B;&#x4ECB;&#x5165;&#x3059;&#x308B;](../.gitbook/assets/cloud-waf.png)

### Cloudflareの導入

クラウド型WAFの検証のため一時的にこのサイトへCloudflareのWAFを導入する。`akaki.io` はGitHub Pagesで提供しているため、本来はGitHubが所有するIPアドレスに名前解決される⁶。今回は `akaki.io` のNSレコードにCloudflareのネームサーバーを設定し、WAFセンターのIPアドレスに名前解決されるように変更する。

![](../.gitbook/assets/dig_ns.png)

![](../.gitbook/assets/dig_a.png)

WAFの導入によりクエリ文字列に [`' or 1 = 1 --`](https://akaki.io/?q=%27%20or%201%20%3D%201%20--) のようなSQLiペイロードを付与したアクセスは遮断される。

![](../.gitbook/assets/cloudflare-waf.png)

## バイパスの仕組み

クラウド型WAFで保護すべきサーバーを「オリジンサーバー」と呼び、そこに割り当てられたグローバルIPアドレスを「オリジンIP」と呼ぶ。オリジンIPへのアクセスがWAFセンター以外にも許可されている場合、インターネットからWAFを経由せずにWebサーバーに直接アクセスできる状態となる。つまり攻撃者は標的サイトのオリジンIPを特定できればWAFに遮断されずに攻撃できる。

![&#x56F3;2. &#x30AA;&#x30EA;&#x30B8;&#x30F3;&#x30B5;&#x30FC;&#x30D0;&#x30FC;&#x3078;&#x306E;&#x76F4;&#x63A5;&#x653B;&#x6483;&#x306F;&#x30AF;&#x30E9;&#x30A6;&#x30C9;&#x578B;WAF&#x3067;&#x9632;&#x3052;&#x306A;&#x3044;](../.gitbook/assets/cloud-waf_bypass.png)

特定したIPアドレスが本当にオリジンIPかどうかを確かめるには、そのIPアドレスにアクセスした際のレスポンスとドメインにアクセスした際のレスポンスを比較する。コンテンツの内容がほぼ一致していればオリジンIPだと断定できる。ブラウザからIPアドレスにアクセスしただけではレスポンスが返らない場合は、バーチャルホストで動作している可能性があるため Hostヘッダを本来のドメインに変更して再確認する。

### Cloudflareのバイパス

GitHub Pagesが使用するIPアドレスの1つである `185.199.108.153` にアクセスしただけではコンテンツは返らない。Hostヘッダの値を `akaki.io` に変更するとドメインにアクセスした際と同じコンテンツが返ってくる。

![](../.gitbook/assets/cloudflare-waf_bypass.png)

オリジンIPに直接アクセスされるとクラウド型WAFでは保護できなくなる。そのためhostsファイルに `185.199.108.153 akaki.io` を追記するなどしてドメインがオリジンIPに名前解決される状態であれば、クエリ文字列に [`' or 1 = 1 --`](https://akaki.io/?q=%27%20or%201%20%3D%201%20--) を付与したアクセスでも遮断されない。攻撃者はクラウド型WAFをバイパスしてこのサイトを攻撃できる。

## オリジンIPの特定方法

GitHub PagesのようにオリジンIPが明示されていなくても、様々な方法によりオリジンIPを特定できる可能性がある。BlackHat USA 2013でIntegralisの研究者が発表した「[DDoS Protection Bypass Techniques](https://www.blackhat.com/us-13/archives.html#Nixon)」と、2016年にKU LeuvenとStony Brook Universityの共同研究チームが発表した論文「[Maneuvering Around Clouds: Bypassing Cloud-based Security Providers](https://distrinet.cs.kuleuven.be/news/2016/BypassingCloud.pdf)」と、いくつかの興味深いブログを参考にオリジンIPの特定方法をまとめる⁷ ⁸ ⁹ ¹⁰。

### 1. サブドメインからの特定

Webサイトをホストしているドメインの多くはサブドメインで別のサービスを提供している。例えば同じサーバー上でメインのサイトを `example.com` で公開し、バーチャルホストでブログサイトを `blog.example.com` で公開し、FTPサーバーを `ftp.example.com` で運用しているようなケースである。メインのサイトだけがクラウド型WAFで保護されている場合、有効なサブドメインとそれに紐付くIPアドレスがわかればオリジンIPを特定できる。

サブドメインを列挙するツールには、一般的なサブドメインを総当たりするNMAPスクリプトの「[dns-brute](https://nmap.org/nsedoc/scripts/dns-brute.html)」や、DNSの逆引きやSSL証明書の内容など複数の観点でサブドメインを列挙するOWASPの「[Amass](https://github.com/OWASP/Amass)」などがある。サブドメインを列挙できるWebサービス「[DNSdumpster](https://dnsdumpster.com/)」や、Comodo CAが提供する証明書検索サービス「[crt.sh](https://crt.sh/)」でのワイルドカード検索（ `%.example.com` ）も活用できる。Twitterなどで「subdomain enumeration」と検索すると他にも様々なツールが見つかる。

### 2. DNSレコードの参照

#### 2.1. 過去のDNSレコード

インターネットに公開されたDNSレコードを記録し続けている「[DNSDB](https://www.farsightsecurity.com/solutions/dnsdb/)」というデータベースがある。クラウド型WAFの導入前にドメインと紐付けられていたIPアドレスが導入後もオリジンサーバーのIPアドレスとして使用されていた場合、DNSレコードの履歴からオリジンIPを特定できる可能性がある。

過去にドメインに紐付いていたIPアドレスは「[SecurityTrails](https://securitytrails.com/dns-trails)」や「[Netcraft](https://toolbar.netcraft.com/site_report)」、「[ViewDNS.info](https://viewdns.info/iphistory/)」などのDNS履歴サービスでも検索できる。また「[Crimeflare](http://www.crimeflare.org:82/cfs.html)」というWebサービスはDNS履歴などをもとにCloudflareが保護しているIPアドレスを検索できる。列挙したサブドメインをDNS履歴サービスで検索してオリジンIPの特定を試みる「[Bypass firewalls by abusing DNS history](https://github.com/vincentcox/bypass-firewalls-by-DNS-history)」というツールもある。

#### 2.2. 他のDNSレコード

ドメインのAレコードはWAFセンターのIPアドレスに置き換えられていても、別のDNSレコードの内容にオリジンIPに紐付く情報が含まれているケースもある。例えばWebサイトとメールサービスが同じサーバー上で運用されている場合、MXレコードで指定されたドメインに紐付くIPアドレスからWebサイトのオリジンIPを特定できる。TXTレコードに含まれるSPFレコードに記載された送信元IPアドレスや、AAAAレコードに残るIPv6アドレスからもオリジンIPを特定できる可能性がある。

### 3. 共通情報による紐付け

#### 3.1. SSL証明書

クラウド型WAFで保護するサイトをWAFベンダー以外で発行したSSL証明書でHTTPS化するには、発行したSSL証明書をWAFセンターに設置する必要がある。オリジンサーバーでも同じ証明書が使用されていた場合、Censysの証明書検索によりオリジンサーバーと紐付けることでオリジンIPを特定できる可能性がある。

CloudflareではBusinessプランから他社の認証局で発行したSSL証明書をWAFセンターに設置できる。CloudflareのWAFセンターに設置されたSSL証明書を使用しているサイトを探索してオリジンIPの特定を試みるツールとして「[CloudFlair](https://github.com/christophetd/CloudFlair)」や「[Cloudsnare](https://gist.github.com/chokepoint/28bed027606c5086ed9eeb274f3b840a)」がある。

#### 3.2. サイト固有の情報

クラウド型WAFで保護されたサイトのtitle要素や独自のHTTPヘッダ、HTMLに含まれるGoogle AnalyticsのトラッキングコードやreCAPTCHAのサイトキーのようなサイト固有の情報をサーチエンジンで検索することで、オリジンサーバーと紐付けられる可能性もある。「[CloudBunny](https://github.com/Warflop/CloudBunny)」というツールはCensysでの証明書検索に加え、Censys、Shodan、ZoomEyeの3つのサーチエンジンでtitle要素を検索してオリジンIPの特定を試みる。

### 4. Webサイトからの取得

#### 4.1. アウトバウンド通信

オリジンサーバーから外部サーバーへ向かう通信はWAFセンターを経由しない。そのためクラウド型WAFで保護されたサイトに外部サーバーへの通信を発生させられる機能が存在する場合、管理下のサーバーに通信を発生させることで接続元IPアドレスからオリジンIPを特定できる。URLから画像を取得する機能や、WebhookやPingbackといった機能は任意のサーバーに通信を発生させられるためオリジンIPの特定に利用できる。

![&#x56F3;3. &#x30A2;&#x30A6;&#x30C8;&#x30D0;&#x30A6;&#x30F3;&#x30C9;&#x901A;&#x4FE1;&#x306F;WAF&#x30BB;&#x30F3;&#x30BF;&#x30FC;&#x3092;&#x7D4C;&#x7531;&#x3057;&#x306A;&#x3044;](../.gitbook/assets/outbound.png)

Webサイトにメール配信機能がありオリジンサーバーからメールが送信される場合、受信したメールのReceivedヘッダに含まれる送信元IPアドレスからオリジンIPを特定できる。アカウント登録完了やパスワード再設定の際に送信されるメールなどから特定できる可能性がある。

#### 4.2. コンテンツ

Webサイトの管理不備により公開状態になったログファイルや開発時のファイル、HTMLコメントにオリジンIPが記載されている可能性がある。またWebサーバーやWebアプリケーションが出力するエラーメッセージの内容にオリジンIPが含まれる可能性もある。

### 5. その他の方法

オリジンIPを特定する方法は他にもある。例えば全てのグローバルIPアドレスにアクセスして確認する力技や、WAFベンダーの社員を装ってサイト運営者からオリジンIPを聞き出すようなソーシャルエンジニアリングが考えられる。他にもメンテナンスやサーバー移行などの理由でクラウド型WAFの利用を一時停止した際に公開されるオリジンIPを見逃さないようにドメインを監視するなど、手間を惜しまなければオリジンIPを特定できる可能性は高い。

過去にはCloudflareのDMCA申し立て対応を悪用することでオリジンIPの特定が可能だった。CloudflareはDMCA侵害の疑いのあるサイトのオリジンIPを申立人に提供していたため、虚偽の申し立てを行なうことで標的サイトのオリジンIPを特定できた。しかしCloudflareはこの対応プロセスを見直し、現在は正当な申し立てであっても申立人にオリジンIPを提供しない仕組みになっている¹¹。

## ツールによる検査

オリジンIPを特定するツールとして取り上げた4つのツールと、KU LeuvenとStony Brook Universityの共同研究チームが開発した「[CloudPiercer](https://cloudpiercer.org/)」というWebサービスを加えた合計5つのツールを使用して `akaki.io` を検査する。CloudPiercerは対象サイトの管理者のみ検査できる仕様であり、今回は `akaki.io` のTXTレコードに認証コードを含めることでサイト管理者であることを証明している。

![](../.gitbook/assets/dig_txt.png)

各ツールでの検査は以下のような結果になった。CloudPiercerだけが `akaki.io` のオリジンIPを特定できた。

| ツール名称 | 特定方法 | 特定可否 | 可否理由 |
| :--- | :--- | :--- | :--- |
| [CloudFlair](https://github.com/christophetd/CloudFlair) | [3.1](cloud-waf_bypass.md#31-ssl証明書) | 否 | `akaki.io` はCloudflareのSSL証明書を使用しているため |
| [Cloudsnare](https://gist.github.com/chokepoint/28bed027606c5086ed9eeb274f3b840a) | [3.1](cloud-waf_bypass.md#31-ssl証明書) | 否 | `akaki.io` はCloudflareのSSL証明書を使用しているため |
| [CloudBunny](https://github.com/Warflop/CloudBunny) | [3.1](cloud-waf_bypass.md#31-ssl証明書) [3.2](cloud-waf_bypass.md#32-サイト固有の情報) | 否 | `akaki.io` はCloudflareのSSL証明書を使用しており、`www.akaki.io` もCloudflareの保護下にあるため |
| [CloudPiercer](https://cloudpiercer.org/) | [1](cloud-waf_bypass.md#1-サブドメインからの特定) [2.1](cloud-waf_bypass.md#21-過去のdnsレコード) [2.2](cloud-waf_bypass.md#22-他のdnsレコード) [3.1](cloud-waf_bypass.md#31-ssl証明書) [4.1](cloud-waf_bypass.md#41-アウトバウンド通信) [4.2](cloud-waf_bypass.md#42-コンテンツ) | 可 | DNS履歴サービスの検索で `akaki.io` の過去のIPアドレスが見つかったため |
| [Bypass firewalls by abusing DNS history](https://github.com/vincentcox/bypass-firewalls-by-DNS-history) | [2.1](cloud-waf_bypass.md#21-過去のdnsレコード) | 否 | DNS履歴サービスの検索で `*.akaki.io` の過去のIPアドレスが見つからなかったため |

CloudPiercerは6種類の方法によりオリジンIPの特定を試みる。今回の検査ではDNS履歴の追跡によりオリジンIPを特定している。CloudPiercerはOSSではないため詳細な検査ロジックは不明だが、論文によるとDNS履歴の追跡には「[DomainTools](https://www.domaintools.com/)」と「[Myip.ms](https://myip.ms/)」を使用しているとのこと。実際にMyip.msで `akaki.io` を検索するとCloudflareの導入前に使用していたIPアドレスが見つかるため、CloudPiercerも同様にオリジンIPを特定したと推測する。

![](../.gitbook/assets/cloudpiercer.png)

## バイパスへの対策

クラウド型WAFのバイパスを防ぐには「オリジンIPの特定阻止」または「オリジンIPへのアクセス制限」が対策となる。前者は数ある特定方法ごとの対策が必要であり、完全にオリジンIPの特定を阻止するのは困難であるため緩和策だと考える。後者はオリジンIPへのアクセスをWAFセンターのみに制限することで、攻撃者は標的サイトのオリジンIPを特定できても直接攻撃できなくなるため、対策として効果が期待できる。

Cloudflareのセキュリティガイドラインでは、オリジンIPの特定を阻止する策として「不要なDNSレコードの削除」や「別サーバーでのメールサービスの提供」、「IPアドレスの変更」をあげている¹²。またオリジンIPへのアクセスを制限する策として「iptablesによるアクセス制限」の手順を説明している¹³。ただしiptablesによるアクセス制限ではサーバーリソースやネットワーク帯域を枯渇させるDDoS攻撃への対策としては不十分な場合がある。より効果的な対策としてCloudflareはWAFセンターとオリジンサーバー間をTLSトンネル化する「[Argo Tunnel](https://www.cloudflare.com/products/argo-tunnel/)」を提供している。

## 所感

CloudflareのWAFを導入するために私は$20を支払ってProプランを契約した。企業向けのプランを利用する場合はより高額な契約料を毎月支払う事になる。高額な契約料を支払ってクラウド型WAFを導入してもバイパスされては意味がない。バグバウンティのスコープでそれを見つけて報告すれば、大抵の企業は報奨金を支払ってくれるだろう。今回まとめた特定方法を総合的にチェックできるツールを使えば効率的に調査できるが、CloudPiercerのようなOSSは見つからなかったので自作するしかない。

オリジンIPを特定する方法は他にもあるだろう。サーバーやミドルウェア起因のリダイレクト処理での漏洩や、SSRFの脆弱性によるアウトバウンド通信により特定できる可能性はある。今後は新しい特定方法を模索していきたい。オリジンIPの特定方法はTorサービスのリアルIPの特定にも応用できそうだが、身の危険を感じる¹⁴ ¹⁵。



¹ [https://blog.christophetd.fr/bypassing-cloudflare-using-internet-wide-scan-data/](https://blog.christophetd.fr/bypassing-cloudflare-using-internet-wide-scan-data/)  
² [https://distrinet.cs.kuleuven.be/news/2016/BypassingCloud.pdf](https://distrinet.cs.kuleuven.be/news/2016/BypassingCloud.pdf)  
³ [https://www.blackhat.com/us-13/archives.html\#Nixon](https://www.blackhat.com/us-13/archives.html#Nixon)  
⁴ [https://blog.cloudflare.com/ddos-prevention-protecting-the-origin/](https://blog.cloudflare.com/ddos-prevention-protecting-the-origin/)  
⁵ [https://blogs.akamai.com/2013/08/bypassing-content-delivery-security.html](https://blogs.akamai.com/2013/08/bypassing-content-delivery-security.html)  
⁶ [https://help.github.com/en/articles/setting-up-an-apex-domain\#configuring-a-records-with-your-dns-provider](https://help.github.com/en/articles/setting-up-an-apex-domain#configuring-a-records-with-your-dns-provider)  
⁷ [https://www.1337pwn.com/how-to-find-real-ip-address-website-behind-cloudflare/](https://www.1337pwn.com/how-to-find-real-ip-address-website-behind-cloudflare/)  
⁸ [https://www.ericzhang.me/resolve-cloudflare-ip-leakage/](https://www.ericzhang.me/resolve-cloudflare-ip-leakage/)  
⁹ [https://www.wordfence.com/blog/2016/10/endpoint-vs-cloud-security-cloud-waf-bypass-problem/](https://www.wordfence.com/blog/2016/10/endpoint-vs-cloud-security-cloud-waf-bypass-problem/)  
¹⁰ [https://www.rootusers.com/find-the-ip-address-of-a-website-behind-cloudflare/](https://www.rootusers.com/find-the-ip-address-of-a-website-behind-cloudflare/)  
¹¹ [https://blog.cloudflare.com/thoughts-on-abuse/](https://blog.cloudflare.com/thoughts-on-abuse/)  
¹² [https://support.cloudflare.com/hc/en-us/articles/200170166-General-website-security-guidelines](https://support.cloudflare.com/hc/en-us/articles/200170166-General-website-security-guidelines)  
¹³ [https://support.cloudflare.com/hc/en-us/articles/200169166-How-do-I-whitelist-Cloudflare-s-IP-addresses-in-iptables-](https://support.cloudflare.com/hc/en-us/articles/200169166-How-do-I-whitelist-Cloudflare-s-IP-addresses-in-iptables-)  
¹⁴ [https://www.secjuice.com/finding-real-ips-of-origin-servers-behind-cloudflare-or-tor/](https://www.secjuice.com/finding-real-ips-of-origin-servers-behind-cloudflare-or-tor/)  
¹⁵ [https://blog.0day.rocks/securing-a-web-hidden-service-89d935ba1c1d](https://blog.0day.rocks/securing-a-web-hidden-service-89d935ba1c1d)

