---
description: 2019年4月20日に神戸デジタル・ラボで開催された大和セキュリティ勉強会「Powershell忍者入門」に参加した。今回の勉強会はセキュリティ分野での関心も高まる「PowerShell」がテーマである。私はWindows環境へのペネトレーション手法を学ぶきっかけを求めて参加した。はじめにPowerShellの概要と基本操作を講義形式で学び、その後は各自でオンライントレーニングに取り組んだ。
---

# 大和セキュリティ勉強会でPowerShellの基礎を学ぶ

<time datetime="2019-04-30">Apr 30, 2019</time>

---

2019年4月20日に神戸デジタル・ラボで開催された大和セキュリティ勉強会「[Powershell忍者入門](https://yamatosecurity.connpass.com/event/126404/)」に参加した。今回の勉強会はセキュリティ分野での関心も高まる「PowerShell」がテーマである。私はWindows環境へのペネトレーション手法を学ぶきっかけを求めて参加した。はじめにPowerShellの概要と基本操作を講義形式で学び、その後は各自でオンライントレーニングに取り組んだ。

## PowerShellの概要

PowerShellは、2006年にMicrosoftがリリースしたコマンドラインシェルおよびスクリプト言語で、Windows 7以降のOSには標準搭載されている。GitHubが2018年に発表した[急成長している言語ランキング](https://github.blog/jp/2018-11-20-state-of-the-octoverse-top-programming-languages/#%e6%80%a5%e6%88%90%e9%95%b7%e3%81%97%e3%81%a6%e3%81%84%e3%82%8b%e8%a8%80%e8%aa%9e%ef%bc%88%e3%82%b3%e3%83%b3%e3%83%88%e3%83%aa%e3%83%93%e3%83%a5%e3%83%bc%e3%82%bf%e6%95%b0%ef%bc%892018%e5%b9%b49)では4位になるなど注目を浴びている。またアメリカのセキュリティ企業であるRed Canaryが2019年に公開したレポートによると、PowerShellは[ATT&CK](https://attack.mitre.org/)の中で最も悪用されているテクニックであり、攻撃者からの人気も高い<sup id="f1">[¹](#fn1)</sup>。

<figure><img src="/assets/2019/learning_powershell/redcanary_graph.webp" width="770" height="97" decoding="async" alt="" /><figcaption>出典：Threat Detection Report 2019 - Red Canary</figcaption></figure>

PowerShellにはMicrosoftが開発した「Windows PowerShell」と、その後オープンソース化した「PowerShell Core」の2種類が存在する。MicrosoftはWindows PowerShellの開発を既に終了し、現在はPowerShell Coreの開発に注力している。しかし利用できるモジュールの多さから攻撃者は現在もWindows PowerShellを使用するケースが多いため、勉強会ではWindows PowerShellからの学習を推奨していた。

Windows PowerShellはバージョン5.0から[Just Enough Administration](https://docs.microsoft.com/ja-jp/powershell/jea/overview)（Windowsにおける `sudo` ）や[Constrained Language Mode](https://devblogs.microsoft.com/powershell/powershell-constrained-language-mode/)（制限モード）、[Antimalware Scan Interface](https://docs.microsoft.com/en-us/windows/desktop/amsi/antimalware-scan-interface-portal)（PowerShellアンチウィルス）などのセキュリティ機能が搭載されている。しかし攻撃に必要な機能はバージョン2.0で搭載されているため、多くの攻撃者が現在もバージョン2.0を使用しているとのこと。

## PowerShellの基本操作

はじめに[Windows PowerShellの起動方法](https://docs.microsoft.com/ja-jp/powershell/scripting/getting-started/starting-windows-powershell)と[バージョンの確認方法](https://docs.microsoft.com/ja-jp/powershell/scripting/install/installing-windows-powershell#how-to-check-the-version-of-powershell)、[PowerShellの特徴](https://docs.microsoft.com/ja-jp/powershell/scripting/learn/understanding-important-powershell-concepts)について学んだ後、PowerShellを実際に動かしながら基本操作を学んでいった。私はVM環境のWindows 10で[Windows PowerShell ISE](https://docs.microsoft.com/ja-jp/powershell/scripting/components/ise/introducing-the-windows-powershell-ise)を起動して動作を確認した。

### PowerShellコマンド

PowerShellのコマンドは `動詞-名詞` の形式で名付けられた[コマンドレット](https://docs.microsoft.com/ja-jp/powershell/scripting/learn/learning-powershell-names#cmdlets-use-verb-noun-names-to-reduce-command-memorization)の他に、[エイリアス](https://docs.microsoft.com/ja-jp/powershell/scripting/learn/using-familiar-command-names)や関数、Windowsに付属するネイティブコマンド（ `ping.exe` のようなEXE形式のコマンド）も含まれる。多くのコマンドレットにはUnix形式、Windows形式、PowerShell形式の3つのエイリアスが設定されている。例えば、指定のディレクトリにあるファイルの一覧を表示する `Get-ChildItem` の場合、`ls`、`dir`、`gci` がエイリアスとして機能する。エイリアスの設定は `Get-Alias` で確認できる。

<figure><img src="/assets/2019/learning_powershell/get-alias.webp" width="770" height="106" decoding="async" alt="" /></figure>

PowerShellでもLinuxと同様にリダイレクト（ `>` や `>>` ）を使用できるが、勉強会では [`Out-*` コマンドレットによるリダイレクト](https://docs.microsoft.com/ja-jp/powershell/scripting/samples/redirecting-data-with-out---cmdlets)を推奨していた。またパイプライン（ `|` ）もLinuxと同様に使用できる。[PowerShellのパイプライン](https://docs.microsoft.com/ja-jp/powershell/scripting/learn/understanding-the-powershell-pipeline)の出力はテキストではなくオブジェクトになるため、`awk` や `sed` のようなコマンドで整形する必要がない。例えば、実行中のプロセスの名前だけをファイルに書き出す場合、以下のようにコマンドを実行すればよい。`Select-Object` で指定したプロパティの情報だけをファイルに書き出せる。

<figure><img src="/assets/2019/learning_powershell/get-process.webp" width="770" height="28" decoding="async" alt="" /></figure>

### PowerShellモジュール

コマンドやスクリプトは[PowerShellモジュール](https://docs.microsoft.com/en-us/powershell/developer/module/understanding-a-windows-powershell-module)というファイルで定義されている。PowerShellモジュールは `$env:PSModulePath` に設定されたディレクトリから読み込まれる。読み込まれたモジュールは `Get-Module` で確認できる。

<figure><img src="/assets/2019/learning_powershell/get-module.webp" width="770" height="127" decoding="async" alt="" /></figure>

新たにモジュールを追加する場合は `Import-Module` を使用する。

### PowerShellスクリプト

PowerShellはスクリプト言語でもあるので、処理の[関数化](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions)や[if文](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_if)、[for文](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_for)によるフロー制御なども可能である。`.ps1` 拡張子のファイルに処理を記載することでスクリプトとして実行できる。また独自関数をコマンドとして利用する場合は `.psm1` 拡張子のファイルで関数を定義して `Import-Module` で追加する。

PowerShellの関数は全ての出力が戻り値として扱われるのが特徴である。例えば以下のような関数の戻り値は、引数1と引数2の和だけでなく `Write-Output` によるメッセージも戻り値に含まれる。

```
Function Add-Numbers([int]$one, [int]$two) {
  Write-Output "What's $($one) and $($two)?"
  return $one + $two
}
```

<figure><img src="/assets/2019/learning_powershell/add-numbers.webp" width="770" height="76" decoding="async" alt="" /></figure>

### PowerShellプロファイル

[PowerShellプロファイル](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_profiles)は自動起動スクリプトであり、Linuxの `.bashrc` のような役割を持つ。攻撃者はPowerShellマルウェアをプロファイルとして登録する可能性があるのでフォレンジックで確認すべきポイントとのこと。以下のコマンドを実行すると設定されているプロファイルを確認できる。

<figure><img src="/assets/2019/learning_powershell/profile.webp" width="770" height="116" decoding="async" alt="" /></figure>

### PowerShellの実行ポリシー

PowerShellには[実行ポリシー](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)という制限機能があり、デフォルトでは第三者のスクリプトを実行できない「[Restricted](https://docs.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_execution_policies#restricted)」が設定されている。署名されていないローカルのスクリプトを実行したりモジュールを追加したりする場合は、PowerShellを管理者権限で起動して `Set-ExecutionPolicy` により実行ポリシーを「[RemoteSigned](https://docs.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_execution_policies#remotesigned)」などに変更する必要がある。

<figure><img src="/assets/2019/learning_powershell/set-executionpolicy.webp" width="770" height="63" decoding="async" alt="" /></figure>

PowerShellの実行ポリシーにはさまざまなバイパス手法が発見されている<sup id="f2">[²](#fn2)</sup>。しかしMicrosoftは実行ポリシーをセキュリティ対策として位置付けていないため<sup id="f3">[³](#fn3)</sup>、完全に修正されておらず現在もバイパスは可能とのこと。

## PowerShellのトレーニング

勉強会の後半は「[Under The Wire](http://underthewire.tech/)」というPowerShellのオンライントレーニングに取り組んだ。トレーニング環境のサーバーにSSHでログインし、前半に学んだPowerShellの知識を活かして問題を解いていく。オンライントレーニングは5つのステージに分かれており、初級ステージである[Century](http://underthewire.tech/century/century.htm)のレベル1の問題から進める。

まずレベル1のユーザー `century1` でサーバーにログインする。レベル1のユーザーはJoeアカウントになっているため、パスワードに `century1` を入力してログインする。

<figure><img src="/assets/2019/learning_powershell/century1.webp" width="770" height="114" decoding="async" alt="" /></figure>

レベル1のユーザーの状態で与えられた問題に挑戦し、その解答がレベル2のユーザーのパスワードとなる。参考としてレベル1の解法を紹介する。レベル1の問題（[Century1](http://underthewire.tech/century/century1.htm)）は以下である。

> The password for Century2 is the build version of the instance of PowerShell installed on this system.

PowerShellのビルドバージョンがレベル2のパスワードらしいので、`$PSVersionTable` を実行してバージョン情報を取得する。

<figure><img src="/assets/2019/learning_powershell/psversion.webp" width="770" height="210" decoding="async" alt="" /></figure>

BuildVersionの値は `10.0.14393.2791` となっている。この値をレベル2のユーザー `century2` のパスワードとして入力するとログインに成功する。

<figure><img src="/assets/2019/learning_powershell/century2.webp" width="770" height="115" decoding="async" alt="" /></figure>

この状態でレベル2の問題（[Century2](http://underthewire.tech/century/century2.htm)）に取り組む。徐々に難易度が上がっていく問題に挑戦し、レベル15まで解答できればステージクリアとなる。私は勉強会の時間内にレベル9まで解答できた。

## 所感

今回の[大和セキュリティ勉強会](https://yamatosecurity.connpass.com/)はPowerShellの基本を学びたい人向けの内容で、PowerShellをほとんど使ったことがない私でも理解できる難易度だった。エイリアスの説明の際に講師のザックさんが「自分は `ls` を使っていても他の人は別のコマンドを使っているかもしれないので、`Get-ChildItem` とそのエイリアス3つを覚える必要がある」と言っていた。自分が使わないから知らなくて良いわけではない。攻撃者の狙いを把握するためにも基本の理解は欠かせないと感じた。

勉強会の中では[WMIの悪用](https://www.peerlyst.com/posts/wmi-wiki-for-offense-and-defense-s-delano)や[Active Directoryの情報収集](https://github.com/PyroTek3/PowerShell-AD-Recon)も話題に上がり、現実世界でのPowerShellの悪用事例に興味が湧いた。実際の攻撃をシミュレートするペネトレーションテストではPowerShellを利用したWindows環境の攻略も求められる。勉強会で教えてもらった[PowerShell Empire](https://www.powershellempire.com/)というペネトレーションツールや[PowerShell Gallery](https://www.powershellgallery.com/)というリポジトリを今後の参考にしたい。

---

<sup id="fn1">[¹](#f1)</sup> https://resources.redcanary.com/hubfs/ThreatDetectionReport-2019.pdf  
<sup id="fn2">[²](#f2)</sup> https://blog.netspi.com/15-ways-to-bypass-the-powershell-execution-policy/  
<sup id="fn3">[³](#f3)</sup> https://technet.microsoft.com/en-us/gg261722.aspx
