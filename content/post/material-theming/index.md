---
title: "Material Themingとは"
date: 2022-09-03T16:42:56+09:00
description: "Material Designでデザインの一貫性とカスタマイズ・保守性を高めるMaterial Themingについて、概要を説明する。"
slug: material-theming
tags:
  - Android
keywords:
  - Android
  - Material Design
  - Material Theming
---

[Material Design](https://material.io/)をもとに作られるアプリに対し、一貫性のあるデザインを維持しつつカスタマイズを可能にするシステムのこと。トーン & マナーを[Material Theming](https://material.io/design/material-theming/overview.html#material-theming)に落とし込むことで、ある属性のデザインを調整したくなった場合（例えば、アプリで用いるボタンの背景色とテキストスタイルを調整したい、など）に、Material Themingに定義された属性に対応する値を調整するだけで、その属性を用いている全てのUIコンポーネントに対する適用が可能となる（個別に調整していく手間が省ける）。

Material Themingでは、大きく分けて次の四つのカテゴリが存在する。

- Color
- Typography
- Shape
- Icons

UIコンポーネントごとに、Colorのこの属性で背景色を表現する、とか、Typographyのこの属性でテキストを表現する、などが定められている。あるUIコンポーネントについて、背景色としてColorのどの属性が参照されるのか、Typographyのどの属性でテキストが表現されるのか、などを知りたければ、Material Designの[Components](https://material.io/components)から対象のUIコンポーネントのページを開き、上部のタブのうち*IMPLEMENTATION*を選び、*Anatomy and key properties*の表を見てみると良い。*Default value*の列に記載されているものが、デフォルトで参照されるMaterial Themingの属性値である。

属性値は全てのUIコンポーネントに影響するが、「Buttonに対してのみ、参照される属性値を変えたい」といった対応や、あるいは「このButtonに対してのみ、参照される属性値を変えたい」といったような対象を絞っての調整も可能である。

デフォルトのMaterial Themingでは表現がしづらい場合は、Themingをカスタマイズすることも検討できる。

- Material Themingを拡張し、属性を追加する
- ColorやTypography、Shapeを、独自の実装に入れ替える
- Material Themingに代わるデザインシステムを構築する

下にいくにつれて、仕組みを用意し適用するための実装コストがかさむ。

Androidアプリを開発する際、どのような形でMaterial Themingを組み込んでいるかに関しては、以下のドキュメントを眺めてみると良い。

[Material Theming in Compose  |  Jetpack Compose  |  Android Developers](https://developer.android.com/jetpack/compose/themes/material)

Material Themingをカスタマイズする際、それぞれでどのような実装が必要となるかに関しては、以下のドキュメントを眺めてみると良い。

[Custom design systems in Compose  |  Jetpack Compose  |  Android Developers](https://developer.android.com/jetpack/compose/themes/custom)

では、Material Themingの四つのカテゴリについて、それぞれ説明していく。

## Color

アプリで用いる色を用途ごとに定義する。デフォルトでは、次に挙げる12個のカテゴリが提供されている。

- Primary
- Primary Variant
- Secondary
- Secondary Variant
- Background
- Surface
- Error
- On Primary
- On Secondary
- On Background
- On Surface
- On Error

アプリのブランドを表すものとして部分的に用いられるものが、PrimaryとSecondaryである。Primaryは[App bars](https://material.io/components/app-bars-top)や[Buttons](https://material.io/components/buttons)の背景色などで用いられる。Secondaryはアクセントカラーとして用いられることが多く、例えば[Floating Action Button](https://material.io/components/buttons-floating-action-button)や選択を促す系統のコンポーネント（[Checkboxes](https://material.io/components/checkboxes)や[Radio buttons](https://material.io/components/radio-buttons)など）などで用いられる。PrimaryとSecondaryには、それぞれのVariantカテゴリも提供されており、PrimaryやSecondaryの色相はそのままで明度や彩度を変え、PrimaryやSecondaryを補うために用いられる（例えば、Top app barとStatus barで色の違いを出すために、Status barにはPrimary Variantが用いられる）。

Backgroundは、スクロール可能な領域の背景色として目にする。Surfaceは、[Cards](https://material.io/components/cards)や[Sheets](https://material.io/components/sheets-bottom)、[Menus](https://material.io/components/menus)、[Dialogs](https://material.io/components/dialogs)などのコンポーネントの背景色として用いられる。Errorは、例えば[Text fields](https://material.io/components/text-fields)でエラーの状態を示す際などに用いられる。

カテゴリ名の先頭に*On*がついているものは、後ろに続くカテゴリの色で背景が表現されているコンポーネントの、テキストやアイコンなどの色として用いられる。

## Typography

アプリで用いるテキストスタイルを用途ごとに定義する。デフォルトでは、次に挙げる13個のカテゴリが提供されている。

- H1 Headline
- H2 Headline
- H3 Headline
- H4 Headline
- H5 Headline
- H6 Headline
- Subtitle 1
- Subtitle 2
- Body 1
- Body 2
- BUTTON
- Caption
- OVERLINE

アプリでテキストを実装する際は、これらカテゴリから一つ選び、スタイルをあてていく。必要であれば、フォントファミリーやフォントウェイト、フォントスタイル（イタリックとか）、フォントサイズなどをカスタマイズできる。

## Shape

コンポーネントの形状を、コンポーネントの大きさごとに定義する。デフォルトでは、小さなコンポーネント、中くらいのコンポーネント、大きなコンポーネントの三つでカテゴリ分けされている。

[https://material.io/design/shape/applying-shape-to-ui.html#shape-scheme](https://material.io/design/shape/applying-shape-to-ui.html#shape-scheme)

コンポーネントの角の形状を変えられ、角を丸くするか、あるいは直線的に角を切り落とせる。どの程度角を丸めるか、あるいはどの程度角を切り落とすかのサイズを指定できる。また、全ての角に対し適用するほか、特定の角のみに適用するとか、特定の角のみサイズを小さく・大きくする、なども可能である。なお、[Shapeの種類を混ぜて適用することはブランドのスタイルを認識しづらくなるということで推奨されていない](https://material.io/design/shape/applying-shape-to-ui.html#picking-shapes)。

## Icons

提供されている[Material Icons](https://fonts.google.com/icons?icon.style=Filled&amp;icon.set=Material+Icons)だが、次に挙げる五つの種類が存在する。

- Filled icons
- Sharp icons
- Rounded icons
- Outlined icons
- Two-tone icons

最近リリースされた[Material Symbols](https://fonts.google.com/icons?icon.set=Material+Symbols)では、次に挙げる三つの種類が存在する。

- Outlined
- Rounded
- Sharp

一つのUIに異なる種類のアイコンを混在させるとデザインの一貫性が損なわれるため、避けること。

---

以上がMaterial Themingの概要である。これを踏まえてアプリをデザインし実装することで、アプリで一貫したデザインとなり、将来にわたっての保守性も高くなる。ぜひ活用したい。

