---
description: 第2回目は倫理的なハッカーとしてのルールを説明します。
---

# 倫理的なハッカーとして守るべきルールとは

<p class="modest" align="left">Sep 5, 2016</p>

---

第2回目は倫理的なハッカーとしてのルールを説明します。

スポーツと同じように、脆弱性を見つける際もルールを守らないと、悪意はないのに不正行為とみなされてしまう可能性があります。不正疑惑をかけられないためにも、脆弱性の調査を始める前にルールをしっかり抑えておきましょう。

## 脆弱性調査による影響

脆弱性を見つけるには開発者が意図していないような操作を行なうため、ソフトウェアの動作に影響を与えることがあります。

調査対象のソフトウェアが、パッケージソフトウェアのようなローカルで動作するものであれば影響を受けるは自身のみです。しかし、Webアプリケーションのようなインターネットを介したリモートで動作するものであった場合、その運営者やユーザーなど他人にまで影響が及ぶ可能性があります。

バグ報奨金制度では各社独自のルールが決められており、それに従うことで法的なトラブルは避けられます。しかし、許可を得ていないサービスを勝手に調査した場合、その影響や調査方法によっては法的な責任を問われる可能性があります。

## 業務妨害罪

調査行為によりサービスの停止やデータを破壊し、他人の業務や活動を妨げてしまうと「業務妨害罪」にあたる可能性があります。この法律には次の3つが含まれています。

* **業務妨害罪**
  * **偽計業務妨害罪（刑法233条）**
  * **威力業務妨害罪（刑法234条）**
  * **電子計算機損壊等業務妨害罪（刑法234条の2）**

「電子計算機損壊等業務妨害罪」での業務妨害にあたる行為をWikipediaでは次のように説明しています。

> 業務に使用するコンピュータの破壊、コンピュータ用のデータの破壊、コンピュータに虚偽のデータや不正な実行をするなどの方法により、コンピュータに目的に沿う動作をしないようにしたり、目的に反する動作をさせたりして、業務を妨害する行為
>
<p align="right"><a href="https://ja.wikipedia.org/wiki/%E4%BF%A1%E7%94%A8%E6%AF%80%E6%90%8D%E7%BD%AA%E3%83%BB%E6%A5%AD%E5%8B%99%E5%A6%A8%E5%AE%B3%E7%BD%AA"><em>Wikipedia「信用毀損罪・業務妨害罪」より抜粋</em></a></p>

過去の事例を見てみましょう。2014年、オンラインゲーム「[サドンアタック](https://web.archive.org/web/20160604011123/https://sa.nexon.co.jp/information/notice.aspx?no=4505)」に対し不正なプログラム（チートツール）を用いて運営会社の業務を妨げたとして、少年3人が電子計算機損壊等業務妨害の疑いで書類送検されました。見つけた脆弱性を悪用してゲームでズル（チート）をすると刑法で裁かれることもあるのです。

また、一時的に大量の調査パターンを試すツール（脆弱性スキャナー）を用いた調査は、ソフトウェアの動作に影響を与える可能性が高くなります。このようなツールは自身が管理するソフトウェア以外に使用してはいけません。

## 不正アクセス禁止法

調査行為による影響だけでなく、調査行為自体が不正アクセスとして法に触れる場合もあります。不正アクセスを罰する法律として次のものがあります。

* **不正アクセス行為の禁止等に関する法律（不正アクセス禁止法）**

この法律での不正アクセスにあたる行為をWikipediaでは次のように説明しています。

> 1. **電気通信回線**（インターネット・LAN等）を通じて、アクセス制御機能を持つ電子計算機にアクセスし、**他人の識別符号**（パスワード・生体認証など）を入力し、アクセス制御機能（認証機能）を作動させて、本来制限されている機能を利用可能な状態にする行為 （1号） 
> 2. **電気通信回線**を通じて、アクセス制御機能を持つ電子計算機にアクセスし、**識別符号以外の情報や指令**を入力し、アクセス制御機能を作動させて、本来制限されている機能を利用可能な状態にする行為 （2号） 
> 3. **電気通信回線**を通じて、アクセス制御機能を持つ**他の電子計算機により制限されている**電子計算機にアクセスし、**識別符号以外の情報や指令**を入力し、アクセス制御機能を作動させて、本来制限されている機能を利用可能な状態にする行為 （3号）
> 
<p align="right"><a href="https://ja.wikipedia.org/wiki/%E4%B8%8D%E6%AD%A3%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E8%A1%8C%E7%82%BA%E3%81%AE%E7%A6%81%E6%AD%A2%E7%AD%89%E3%81%AB%E9%96%A2%E3%81%99%E3%82%8B%E6%B3%95%E5%BE%8B"><em>Wikipedia「不正アクセス行為の禁止等に関する法律」より抜粋</em></a></p>

他人のパスワードを用いたり脆弱性を悪用したりして、本来権限のないデータや機能にアクセスする行為は「不正アクセス禁止法」に触れ、逮捕される可能性があります。

調査行為が不正アクセス禁止法違反として認められた事例として、2003年に元大学研究員が行なったACCS（一般社団法人 コンピュータソフトウェア著作権協会）のWebサイトへの脆弱性調査があげられます。この事件では、Webサイトで見つけた脆弱性を用いて本来アクセスできないはずのサイト利用者の個人情報を含んだログファイルにアクセスしたとして、元研究員に懲役8ヵ月・執行猶予3年の判決が言い渡されました。

調査行為のすべてが不正アクセスにあたるわけではありませんが、場合によってはこのような刑罰を受けることもあるのです。また、調査行為が不正アクセスにあたるかどうかの判断にはIT技術と法律についての深い知識が求められるため、独自で判断するのは危険です。許可を得ていないサービスへの調査は行なわないようにしましょう。

<br>

脆弱性を見つけることの目的を見失わず、ルールを守って調査することが倫理的なハッカーになるための必須条件です。

次回からはWebアプリケーションの脆弱性の見つけ方を紹介していきます。Webアプリケーションは日常生活や仕事に欠かせないサービスです。その脆弱性を見つけられるスキルが今、倫理的なハッカーに求められています。
