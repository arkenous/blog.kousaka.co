---
title: "iOSDC 2019 1日目 参加ログ"
date: 2019-09-08T16:00:00+09:00
description: "iOSDC 2019 (#iosdc) の1日目（2019/9/6）参加ログ。"
slug: iosdc-2019-day1
categories:
  - Event
---

[iOSDC 2019](https://iosdc.jp/2019/) ([`#iosdc`](https://twitter.com/hashtag/iosdc)) の1日目（2019/9/6）参加ログ。

聞いたセッションは[事前ブログ](https://blog.arkenous.net/iosdc-2019-pre)で書いたものからちょっと変更した。

- 縦書きエディタを6プラットフォームで開発してみて
- Swiftクリーンコードアドベンチャー　~日々の苦悩を乗り越え、確かな選択をするために~
- BLEでiOS/Android間でそこそこ大きなサイズのデータ通信を実現する (L2CAPもあるよ)
- iOSアプリのリジェクトリスクを早期に発見するための取り組み
- Track B LT

あとは茶会にも参加した。

## 縦書きエディタを6プラットフォームで開発してみて

[縦書きエディタを6プラットフォームで開発してみて / On development of multi-platform text editor for vertical writing - Speaker Deck](https://speakerdeck.com/cc4966/on-development-of-multi-platform-text-editor-for-vertical-writing)

[六々](https://twitter.com/496_)さんによる発表。

まず、個人的な趣味で、6プラットフォームを対象に10年も縦書きエディタの開発を続けてらしたのが凄い…と感じた。

テキストを表示する方法として文字の方向を示す「書字方向」と改行の方向を示す「改行方向」という用語も知らなかったし、モンゴル語が左から右に改行していくこととか、牛耕式という右から左、左から右、と交互に書字方向が変わるものもあると知れたのは面白かった。

デスクトップ3プラットフォームにおいてはC++でwxWidgetsを用いて開発してるってのは、まぁそりゃそうかという感じだったが、そこからモバイル3プラットフォーム開発にあたってGUIの部分はネイティブに頼りつつテキスト編集やテキスト表示のコア機能をデスクトップ開発で培ったC++の資産を活かしたということについては、C++強いなーと実感した。どの環境でも使えるのほんと便利。

テキストを表示するためにはテキストを示すバイト列を画像にしないといけないが、ここで文字コード（Unicode）とフォント（OpenType）を用いて変換する手順を丁寧にステップを踏んで説明されていてすごく分かりやすかった。縦書きで表示する際に句読点の表示位置を右寄せするための専用グリフがあるとかアルファベットを回転させないといけないとか、考えることめっちゃ多いなと…。

文字編集にしてもIME APIで縦書き対応のプラットフォーム差異があるとかめっちゃ辛そう…。考えることが山ほどあるのに、その上モバイルプラットフォームごとのテキスト編集のUIもしっかり考えているのはホント凄いと思う。

## Swiftクリーンコードアドベンチャー ~日々の苦悩を乗り越え、確かな選択をするために~

[swift_clean_code.pdf - Speaker Deck](https://speakerdeck.com/shiz/swift-clean-code)

[shiz](https://twitter.com/stzn3)さんによる発表。

クリーンコードとは「分かりやすく」「安全であり」「変更に強いこと」であるが、これをどう実現し継続していくのかについてshizさん自身の考えを述べられていた。

WWDCのセッションで語られた「Start With a Protocol」ってのは全てプロトコルから始めよ、ではなくまず目の前にある個々の具体的な問題を解決し、有効だと判断できたタイミングでGenericな実装にしていくアプローチが良いという解釈であり、これに基づいてコード例を示してGenericにしていく手順を解説されていて分かりやすかった。

「ボーイスカウトの規則」をコーディングに適用し、コードを継続的に改善していくプロ意識を持つってのはいい言葉だなーと思った。チェックアウトした時よりも綺麗にしてチェックイン。これは僕自身も一層肝に銘じて取り組みたいと思った。

あと、同僚にクリーンなコードじゃなくてもいいでしょ？という人がいた場合どう対応するのかという答えとして、クリーンなコードが奨励されるような環境に自分を移すって答えられていて、確かにそれも選択肢の一つだよなと笑いつつも思ってしまった。

全体を通してエモい発表。もっとクリーンなコードを書いていこうという気持ちになった。

## BLEでiOS/Android間でそこそこ大きなサイズのデータ通信を実現する (L2CAPもあるよ)

[iOSDC 2019 BLE - Speaker Deck](https://speakerdeck.com/coe/iosdc-2019-ble)

[日向強](https://twitter.com/coffeegyunyu)さんによる発表。

BLE L2CAPの効率的なデータ通信についてiOSとAndroidの双方のAPIを示しつつ詳細に解説いただいたセッション。触れ込みの通りこのセッションを聞けばBLE、L2CAPのハイパフォーマンスが実装ができるなと感じた。

Exchange MTUとかはちゃんとその存在を知ってないと、iOSだとこれをよしなにしてくれるのに対してAndroidは明示的にしないといけないからAndroidでパフォーマンス悪いのなんでや！？ってなるだろうし、このセッションを聞いて良かった。AndroidのrequestMtuに設定する適切な数値は**517**という貴重な情報も得られたし。iOSでCentralからPeripheralへの最大書き込みサイズを知るための`maximumWriteValueLength(for:)`がいかなる端末であってもiPhone XS Maxの値しか返さないとかいう落とし穴を知ることもできたし。

最後の方でよくあるエラーとその対策も説明いただけたので、Bluetoothで双方向通信の実装をする機会があったらこのスライドを参照しようと思う。

また、ちょいちょい笑いを狙いにいっているのもかなり良かった。これのおかげで話にグッと引き込まれた感がある。

だいぶ昔に端末間でデータのやり取りを行うようなプログラムを趣味で書いていた時に、Bluetoothを使おうとしたけど通信速度が遅すぎて結局Wi-Fi経由でのLAN通信にしたちょっぴり苦い記憶を思い出した。BLEなら20bytesのところをL2CAPなら2048byteを1フレームあたり送れるということなので、できることがグッと広がりそうだなと思った。

## iOSアプリのリジェクトリスクを早期に発見するための取り組み

[iOSアプリのリジェクトリスクを早期に発見するための取り組み - Speaker Deck](https://speakerdeck.com/kesin11/iosapurifalseriziekutorisukuwozao-qi-nifa-jian-surutamefalsequ-rizu-mi)

[Kesin11](https://twitter.com/Kesin11)さんによる発表。

まずQAチームでリジェクトリスクの調査をしてるって時点で強いと思った。

うちだと作ったアプリに対してエンジニア側に知見があるか、もしくはエンジニアが社内の情報蓄積プラットフォームを調べてヤバイのが該当していればPLに対して情報共有するようなエンジニアの技量に依存した状態になっていると思ってる。だけど発表中にあったようなCIで自動検査してエラーがあればビルド自体を落とすみたいな仕組みが整備されていれば、リジェクトによる手戻り発生とかエンジニアの負担増とか、顧客にかける迷惑を軽減できてかなり良さそうだなと思った。

検査手法はInfo.plistの中を見て値を見るとか、JSONをパースするとか割と素直でゴリ押しな感があるけど、どのファイルのどこを見ればどういったリジェクトリスクを調べられるかを説明してくださったので、時間ができたら自前でプラグインを作ってみるもの良さそうだなと思った。

社内にSWETチームがいらっしゃるのってホント羨ましい。

### 茶会

1日目の最後には茶会があった。僕自身はアルコールを飲めなくはないけど（パッチテストしたら飲んじゃいけない側に判定されるかも…）、弱い方だと思う。なのでビールはデプロイされてたけどメインはソフトドリンクなこの会は本当にありがたかった。

ぜひ次回以降も継続していって欲しいし、他の勉強会でもこれを取り入れて欲しいなーと思う。
