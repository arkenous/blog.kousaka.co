---
title: "Material Design 3でのMaterial Theming"
date: 2022-09-04T16:02:50+09:00
description: "Material Design 3におけるMaterial Themingの概要を記載する。"
slug: material-theming-on-material3
tags:
  - Android
keywords:
  - Android
  - Material Design
  - Material Theming
---

PersonalizationのためのMaterial You (Dynamic color) が導入されたことなどから、[Material Design 3](https://m3.material.io/)（以下、Material3）によってMaterial Theming周りも大きく変更された。

とはいえ、Material ThemingがColor, Typography, Shape, Iconsで構成されているという点は変更されていないので、[Material2時点の記事](https://blog.arkenous.net/material-theming)をベースにMaterial3の内容で改めてMaterial Themingを整理する。

Material3で定義されているUIコンポーネントの一覧は[Components](https://m3.material.io/components)ページから参照できる。UIコンポーネントの背景色にどのRoleが参照されるか等は、各UIコンポーネントのページ上部にある*Specs*から確認できる。

Material3でも、従来と同様に「Buttonに対してのみ、参照されるRoleを変えたい」といった対応や、あるいは「このButtonに対してのみ、参照されるRoleを変えたい」といったように対象を絞っての調整が可能である。

（以下の事項は記載を省いています）

- 画像に対するDynamic color

## Color

Material3では、基本となる五つの色を定義することで、それらの色それぞれのトーンパレット（元とする色に白や黒を混ぜたカラーパレットで、13段階の色が得られる）が生成され、ここから*On Primary*や*Secondary Container*といったようなRoleごとの色が自動生成される。

Color schemeを生成する際は、[Webアプリとして提供されているもの](https://m3.material.io/theme-builder#/custom)を用いるか、あるいは[Figma pluginとして提供されているもの](https://www.figma.com/community/plugin/1034969338659738588)を用いる。生成されたColor schemeをAndroidアプリで導入できる形式にExportする機能も備わっている。

### Color scheme

基本となる五つの色は*Accent colors*と*Neutral colors*に分けられている。

*Accent colors*には次の三つが含まれる。

- Primary Key Color
- Secondary Key Color
- Tertiary Key Color

*Neutral colors*には次の二つが含まれる。

- Neutral Key Color
- Neutral Variant Key Color

アプリのブランドを示すものとして、UIにおける重要なコンポーネントに対し用いられるのが*Accent colors*である。新たに追加された*Tertiary Key Color*は、アプリにおける色の表現をさらに広げるために活用できる。

アプリUIの表面や背景、あるいはテキストやアイコンを目立たせるための色として、*Neutral colors*が用いられる。*Neutral Variant Key Color*は、ほどほどに目立たせたいテキストやアイコンや、少し異なる色合いでアプリUIの表面を描きたいとき、またUIコンポーネントの境界線を描画する際に用いられる。

*Accent colors*と*Neutral colors*に加えて、エラーを表す*Error colors*も用意されている。こちらも同様に、基本とする色から*On Error*や*Error Container*、*On Error Container*のRoleに対応する色が自動生成される。

これらの色とは別に、必要であれば*Custom colors*として色を追加できる。これに関しても、他のものと同様に基本とする色を提示し、*On Color*や*Color Container*、*On Color Container*は自動生成される形となる。

Material You (Dynamic color) で、ユーザが設定した壁紙からColor schemeが生成された場合や、アプリで表示するコンテンツからColor schemeが生成された場合などでは、基本となる五つの色とErrorから生成されるColor schemeが影響を受ける。*Custom colors*は影響を受けないため、色相に意味があり、色相を勝手に変えられると困る場合は*Custom colors*を活用すると良い。Dynamic colorで生成された色をもとに、*Custom colors*の色相を微調整し、アプリUI全体で調和を持たせることもできる（Harmonization）。

[https://m3.material.io/styles/color/the-color-system/key-colors-tones](https://m3.material.io/styles/color/the-color-system/key-colors-tones)

[https://m3.material.io/styles/color/the-color-system/custom-colors](https://m3.material.io/styles/color/the-color-system/custom-colors)

### Color roles

*Accent colors*で生成されるRoleのうち、*Container*とつくものは、つかないものに比べてそこまで強調する必要が薄いUIコンポーネントに対して用いられる。

*Neutral colors*から生成されるRoleのうち、*Surface*はElevation levelに応じてPrimary colorによる色が付加される。Elevation levelは+1から+5まであり、それぞれ次のように付加される。

- Elevation +1の場合：5% opacityのPrimary colorが付加される
- Elevation +2の場合：8% opacityのPrimary colorが付加される
- Elevation +3の場合：11% opacityのPrimary colorが付加される
- Elevation +4の場合：12% opacityのPrimary colorが付加される
- Elevation +5の場合：14% opacityのPrimary colorが付加される

*Inverse Surface*、*Inverse On Surface*、*Inverse Primary*といったInverseが付くRoleも用意されており、これらは状況に応じてユーザの注意を惹く必要のあるUIコンポーネント、例えばSnackbarsやTooltipsなどで用いられる。

*Outline* RoleはUIコンポーネントの境界線を明確にするために用いられる。コントラスト比が高くなってしまうため、複数の要素が含まれるCardsのようなUIコンポーネントや、区切り線として用いてはいけない。これらに対しては、*Outline variant color*を用いる必要がある。

*Outline variant color*はコントラスト比が求められない、区切り線やUIコンポーネントの修飾の場面で用いられる。Chipsのような、他のUIコンポーネントとの距離が離れておらず密集度が高い場面や、UIコンポーネントごとの境界をはっきり示す必要がある場面では、一定のコントラスト比を保つために*Outline*を用いる必要がある。

[https://m3.material.io/styles/color/the-color-system/color-roles](https://m3.material.io/styles/color/the-color-system/color-roles)

## Typography

アプリで用いるテキストスタイルを用途ごとに定義する。スタイルは大きく分けて次の五つがある。

- Display
- Headline
- Title
- Body
- Label

これらそれぞれで、大きさごとにLarge, Medium, Smallの三つのScaleが存在するため、あわせて15個のRoleが存在する。

- Display Large
- Display Medium
- Display Small
- Headline Large
- Headline Medium
- …

それぞれのRoleごとの、フォントの種別やフォントサイズ、フォントウェイトなどは次のページに整理されている。もちろん、ここからカスタマイズすることも可能である。

[https://m3.material.io/styles/typography/type-scale-tokens](https://m3.material.io/styles/typography/type-scale-tokens)

## Shape

Material2ではUIコンポーネントのサイズに依存してShapeのサイズが定義されていた。Material3では、UIコンポーネントのサイズに依存しなくなり、単純にShape sizeごとにstyleが定義される形となった。Typographyと似た形。

- None
- Extra small
- Small
- Medium
- Large
- Extra large
- Full

どのUIコンポーネントがどのShapeを用いているかは次のページに整理されている。

[https://m3.material.io/styles/shape/shape-scale-tokens](https://m3.material.io/styles/shape/shape-scale-tokens)

Shapeの形状はMaterial2と同様にRounded cornerとCut cornerの二種類となる。

## Icons

[Material Symbols](https://fonts.google.com/icons?icon.set=Material+Symbols)のみの説明となっている。種類は以下の三つ。

- Outlined
- Rounded
- Sharp

[https://m3.material.io/styles/icons/overview](https://m3.material.io/styles/icons/overview)

---

以上がMaterial3でのMaterial Themingの概要である。Personalizationを意識したものとなっており、特にMaterial You (Dynamic color) あたりをアプリで対応させることで、よりユーザに愛されるアプリとなると思う。また、TypographyやShapeのStyle区分については、Material2のものと比較してより洗練された印象を受ける。

Material3の採用はまだ時期尚早だと思っているが、しっかり状況をチェックしておきたい。

