---
title: "Kotlin Fest 2019 聞いてきた"
date: 2019-09-01T21:00:00+09:00
description: "Kotlin Fest 2019に参加してきたので、メモ。"
slug: kotlinfest-2019
categories:
  - Event
image: "20191215030455-1.webp"
---

2019年8月24日（土）に品川にて開催された、Kotlinを愛でる[Kotlin Fest 2019](https://kotlin.connpass.com/event/129860/)（[`#kotlinfest`](https://twitter.com/hashtag/kotlinfest)）に参加してきた。開催は今回で2回目。セッション傾向としては、Kotlinの言語機能やライブラリ周りのものが4割、サーバサイドKotlinが4割、残り1割ずつでAndroid KotlinとKotlin Multiplatform Project、といった感じだった。セッションの割合からも見て取れるように、Androidアプリ開発のためだけのKotlinではなく、サーバサイドでのKotlinの活用やKotlin Multiplatform Project ( a.k.a. MPP）によるAndroid、iOS、Webといった多様なプラットフォームをKotlinで書く、ということに注目が集まっていると感じた。

僕自身は、数ある素晴らしいセッションのうち、以下のものを聞いてきた。

- オープニングセッション
- Kotlinの型実践入門
- 改めて学ぶContracts
- Kotlin Multiplatform Project入門
- Deep Dive into Kotlin DSL
- LT大会

僕は現在、Androidのアプリ開発にKotlinを用いているので、Kotlinでより綺麗なコードを書く、もっと快適にコーディングするためにどのような技術があるのかを少しでも知るために、上記セッションを選んだ。

## オープニングセッション

[What’s new in Kotlin? - Speaker Deck](https://speakerdeck.com/svtk/whats-new-in-kotlin)

オープニングセッションでは主催の[長澤太郎](https://twitter.com/ngsw_taro)さんによるイベント説明、趣旨の説明があったのち、[Kotlin in Action](https://books.apple.com/jp/book/kotlin-in-action/id1214721178)の著者の一人でありJetBrains社の中の人である[Svetlana Isakova](https://twitter.com/sveta_isakova)さんによるKotlinに追加された・追加される新機能たちの紹介をいただいた。

Kotlinの言語アップデートは快適であることを重視しており（例えばIDEによる自動マイグレーション）、またコミュニティを尊重しオープンであることを説明されていた（[GitHub - Kotlin/KEEP: Kotlin Evolution and Enhancement Process](https://github.com/kotlin/KEEP)）。

Kotlinで現在Experimentalとなっている新機能の説明では、Inline ClassesやContracts、Immutable Collections、Flows、MPPのそれぞれを適宜簡単なコードサンプルを交えつつ分かりやすく説明されていた。後に控えている別の方のセッションを食ってしまうような内容も多く、他のスピーカー泣かせだなぁ…と。僕自身は話を聞く機会が増えて理解が深まるので良かった。

話されていた新機能の中では、ContractsやFlows、MPPについて特に興味を持つことができた。`isNullOrEmpty`でSmart Castsが効かなくてダサいコードを書いていたのが解消される、関数に契約を付けてコンパイラにヒントを与えてよりスマートなコードを書けるようになるってのはワクワクするし、FlowsによってCoroutinesにReactive Streamの思想を持ち込めるってのも面白そうだし、MPPを使ってサーバサイド・iOS・Androidで共通のコードをKotlinで書いてしまえばプラットフォーム毎のチーム間の認識差異による実装ミスを減らせて幸せになれそうだなーと思った。仕事してて稀によくある話なので…。

暇なときに[KEEP: Kotlin Evolution and Enhancement Process](https://github.com/kotlin/KEEP)をチラッと眺めてみるのも面白そうかなと思った。

## Kotlinの型実践入門

[Kotlin Fest 2019: Kotlin型実践入門 - Speaker Deck](https://speakerdeck.com/satoshun/kotlin-fest-2019-kotlinxing-shi-jian-ru-men)

[佐藤隼](https://twitter.com/stsn_jp)さんによる、Kotlinにおける型システムについてのセッション。

Kotlinでプログラムを書く上での魅力の一つである、冗長な型キャストを軽減するSmart Castsだが、今までは`isNullOrEmpty`でnullチェックを判定したとしても、ブロック内では依然としてnullableとして扱われていた。これが、Kotlin 1.3でExperimentalとして導入されているContractsによって関数の性質をプログラマ側で書いてコンパイラにヒントを与えられるようになり、先ほど書いた問題を解決できるなど、もっと賢いコーディングができるようになると紹介されていた。他にも`callsInPlace`で引数として渡されたブロックが関数内で必ず一度だけ呼ばれるといったような様々な記法があるらしい。Smart Castsの機能を拡張してもっと便利に使えるので個人的には使ってみたいなと思う反面、これを組み込む際は契約に反するコードを実装しないような注意深さが求められるように感じ、テストコードで関数の動きが契約に沿っているかを可能な限り保証してやらないといけないのかなと思った。なんにせよ、まだContractsはExperimentalなのでStableになるのが楽しみ。

Nothing型についても、今までよく知らなかったなーと思った。全てのKotlinクラスのサブクラスであり、値が存在しないことを示す型。正常終了することがない関数（throwするだけ、みたいな）の返り値として用いることで、全てのKotlinクラスのサブクラスである性質からエラー判定後の仕方なく書いていたキャストコードや、whenブロック後の到達しないreturnを書かなくて良くなる。使える場面があればどんどん使っていこう。

あとはKotlinにおけるGenericsについてもかなり詳しく分かりやすく解説されていて勉強になった。共変・反変・不変とか、これをKotlinで実装する方法とか。僕自身この辺りを必要に迫られた時になんとなく使っているだけなので、今後型変数を用いる際はこのスライドを参考に実装しようと思う。あと、Javaを触っていた頃はランタイムで型変数にアクセスできなくて辛い思いをしたことがあった気がするけど、Kotlinはinline functionでReified type parametersを使うことで型変数にアクセスできるということで感動した。今後型パラメータの型検査とかをする機会があるかは分からないけど、しっかり頭の片隅に置いておきたい。

## 改めて学ぶContracts

[改めて学ぶContracts - Speaker Deck](https://speakerdeck.com/tommykw/gai-metexue-hucontracts)

[富田健二](https://twitter.com/@tommykw)さんによる、Kotlin 1.3にてExperimentalで導入されたContractsについてのセッション。

関数についての契約をContractsによって定義し、コンパイラにヒントを与えることができる。静的解析も可能なのでIDE側でもContractsを解釈して警告表示を制御してくれて最高。

Run blocking関数だと、それに渡したlambda blockは内部で一回しか呼ばないので、それを示すContractsを宣言してコンパイラにアドバイスする。すると実装時にblock外部で宣言したvalをrun block内部で初期化するようなコードが実装可能となる。

今まで`isNullOrEmpty`でSmart Castsが効かないとかといった問題があったが、Contractsの登場によってこれらが解消され、より人間の目で見て自然なコードの実装が可能となった。

Kotlin 1.3でリリースされたExperimenalな機能。stblibでよく使われているが現時点でもまだExperimentalであり、Core Issueが残っているためプロダクションでの利用はまだできないかな？といった感触らしい。

個人的には自前でContractsを実装するかはともかくとして、コンパイラにヒントを与えるとか面白そうなので待ち遠しい機能。

## Kotlin Multiplatform Project入門

[Kotlin Multiplatform Project入門/Introduction-Kotlin-MPP - Speaker Deck](https://speakerdeck.com/aakira/introduction-kotlin-mpp)

[荒谷光](https://twitter.com/_a_akira)さんによる、Kotlin Multiplatform Project（a.k.a. MPP）についてのセッション。

MPPではAndroidやiOSだけではなく、サーバサイドに対しても共通化したロジックコードを吐き出すことができる。

例えばDomain Objectでnullableなプロパティが存在する場合、Swaggerとかでnullableか否かを書くだけだと、仕様の認識漏れとか実装ミスとかでアプリをNPEで落としてしまう可能性はゼロにできない。

ここでMPPを用いることによって、Domain Objectを全プラットフォームで共通として実装してしまえば、このような仕様のすり合わせをして都度確認するとか、認識相違でバグを生んでしまうリスクを軽減できる。

Android、iOS、Webと複数プラットフォームにまたがるプロジェクトのAndroid側やiOS側を担当することが多いけれど、プラットフォーム間の仕様すり合わせとか認識確認とかでまぁまぁの手間と苦労を感じているので、各プラットフォームを担当される方それぞれにKotlinとGradle（Groovy）を教えてMPPを布教し、Domain Objectとか認証系とかをコードレベルで共通化し、それを利用して他の実装を進めるような形を作れたら余計なバグを作らずに済んで誰もが幸せになれるのになーと思った。

## Deep Dive into Kotlin DSL

[日本語注釈つき Deep Dive into Kotlin DSL - Speaker Deck](https://speakerdeck.com/jmatsu/ri-ben-yu-zhu-shi-tuki-deep-dive-into-kotlin-dsl)

[Jumpei Matsuda](https://twitter.com/@red_fat_daruma)さんによる、DSLとは何？といった話から、KotlinでDSLを書く上でベースとなる（有用な）言語機能などを実際に活用されているライブラリのコードを例示しつつ説明されていた。

普段のコーディングで演算子オーバーロードを書くことはなかなか無かったけど、DSLで利用するのはメソッド名を短くできて直感的で確かに良さそうだなーとか、中置記法使ったらこんな綺麗に、流暢な英語で書けるんやーといった学びがあり、DSLを書きたくなる、もっとスマートに書きたくなるTipsが詰まっていた。Kotlinの持つ機能・記法を深く深く理解していけばもっともっと綺麗にコードを書ける、可読性が高くメンテしやすいコードが書けるというモチベーションを持てるセッションだった。今すぐ仕事で活かしていきたい。

## LT大会

上の各セッションについての所感を書くだけで力尽きたので、LTについてはスライドを貼っておくだけにする…。余力が生まれたら書く。きっと…。

[Kotlin Serializationことはじめ](https://speakerdeck.com/slme/kotlin-serializationkotohazime)

[Jetpack Compose ことはじめ / the beginning of Jetpack Compose - Speaker Deck](https://speakerdeck.com/tkhs0604/the-beginning-of-jetpack-compose)

[静的解析ツール detekt で任意の条件で警告させる - Speaker Deck](https://speakerdeck.com/duck8823/jing-de-jie-xi-turu-detekt-deren-yi-falsetiao-jian-dejing-gao-saseru)

[Kotlin/JS の仕組み / How KotlinJS works - Speaker Deck](https://speakerdeck.com/yukukotani/how-kotlinjs-works)

[GraphQLサーバーを作って味わった天国と地獄 - Speaker Deck](https://speakerdeck.com/ryohysk/graphqlsabawozuo-tutewei-watutatian-guo-todi-yu)

[KotlinTest で始める Property-based Testing/kotlintest-property-based-testing - Speaker Deck](https://speakerdeck.com/yusukehosonuma/kotlintest-property-based-testing)

[try-catchからrunCatchingに_移行した話.pdf - Speaker Deck](https://speakerdeck.com/offwhiite/try-catchkararuncatchingni-yi-xing-sitahua)

## その他

お昼ご飯は個人個人でってことだったのでどうしようかな〜と考えていたけど、会場の下の階にカレー、サラダ食べ放題で¥500とかいう恐ろしいお店があったのでそこでご馳走になった。イベント参加者が次々吸い込まれてた。

![カレーライスとサラダ](20191215030838.webp)

