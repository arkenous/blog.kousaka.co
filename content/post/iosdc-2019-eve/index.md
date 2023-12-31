---
title: "iOSDC 2019 前夜祭 参加したログ"
date: 2019-09-06T01:00:00+09:00
description: "iOSDC 2019 前夜祭に参加してきたので、ログを簡単に残す。"
slug: iosdc-2019-eve
categories:
  - Event
image: "20191215031936.webp"
---

[iOSDC 2019](https://iosdc.jp/2019/) ([`#iosdc`](https://twitter.com/hashtag/iosdc)) の前夜祭（2019/9/5）に参加してきた。

早朝に新神戸からの新幹線で2時間半ほど、そっから東京の職場で仕事した後の17:00からの参加だったので割とキツかった…。

聞いたセッションは[事前ブログ](https://blog.arkenous.net/iosdc-2019-pre)で書いた通り以下の二つ。

- SwiftのStringの文字数の数え方を完全理解する
- 普通に書くと即メモリーリーク！こんなに大変だけど俺はXamarin.iOSを使い続けるぜ！

## SwiftのStringの文字数の数え方を完全理解する。

[SwiftのStringの文字の数え方を完全理解する - Speaker Deck](https://speakerdeck.com/taka1068/swiftfalsestringfalsewen-zi-falseshu-efang-wowan-quan-li-jie-suru)

[Takanori Hirobe](https://twitter.com/taka1068)さんの発表。SwiftのStringはIntでの添字アクセスが出来なかったりCharacterの差し替えを添字で行えなかったりするけど、それはどうしてなのかをUnicodeの考え方を詳細に、なおかつ分かりやすく交えつつ説明されていた。

「ぱ」という文字でもUnicode Scalar的には2要素から成っていたりして、次の要素と結合して初めて一文字（拡張書記素クラスタ）となるかを見ないといけないから、添字にIntを指定する形だと計算量O(N)となってCollectionの添字アクセスO(1)の要請に準拠できずString.Indexを使わないといけないとか、文字の差し替えでも3バイト幅の文字が1バイト幅の文字に変わると後ろの文字のバイトを移動しないといけないから、Intの添字でのsetでO(1)を要請しているMutableCollectionに準拠しないとか、めっちゃ腑に落ちた。

スライドのアニメーションもしっかり活用されててかなり分かりやすかったと思う。いい発表だった。

## 普通に書くと即メモリーリーク！こんなに大変だけど俺はXamarin.iOSを使い続けるぜ！

[Tomohiro Suzuki](https://twitter.com/hiro128_777)さんの発表。前のセッションと若干かぶっていたので途中からの参加。僕自身はXamarin.iOSを触ったことがないけど、社内でXamarinを使っている方、仕事としてこなしている方がいたり、FlutterやKotlin MPPなどマルチプラットフォームな開発が話題になりつつあってちょっと興味があったので聞いた（と言いつつ沼な話に興味があった）。

途中からだったのであまり理解はできなかったけど、GCの動き方とか、ネイティブオブジェクト - マネージオブジェクト間の参照がどうなってるかをしっかり管理しないと即座にメモリリークするとか辛そうな雰囲気を感じた。

セッション中では実際にサンプルコードを示して動かしつつ、カジュアルにメモリリークを起こしていたのが分かりやすく、また面白かった。

メモリリークさせないためには、以下の二つを徹底しなければならない。

- イベントをアタッチしたら、必ずデタッチすること
- Objective-Cの流儀に則り、Selectorを使うこと

「楽したい動機でXamarinを使うと大抵ひどい目にあう。効果と問題点を理解して使うプロ向けツール」というセッション中の言葉に全てが詰まっている気がした。

工数削減のために、楽するためにXamarinを使おうって話が仕事で出ないとは限らないので、その時はこのセッションを思い出して「本当にいいんですか？」って説得の材料に使えそうだなって思った。

## 明日は

明日はいよいよ1日目。以下のセッションを聞こうと思う。

- 縦書きエディタを6プラットフォームで開発してみて
- Swiftクリーンコードアドベンチャー　日々の苦悩を乗り越え、確かな選択をするために
- 画像処理における、UIImageとCGImageとCIImageの効果的な使い分け
- FatViewControllerを安全に書き換える方法が見つからなかったので、どういう痛みを許容するか考えた
- LT

お茶会があるのでそっちも楽しみ。というわけで前夜祭ログ書いたのでさっさと寝る😴

