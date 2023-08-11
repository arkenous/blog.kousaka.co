---
title: "SnapKit備忘録"
date: 2020-06-29T15:03:00+09:00
description: "iOS/macOSのAuto Layout実装をより簡便に行える、SnapKitライブラリに関する備忘録を残す。"
slug: snapkit
tags:
  - iOS
---

[SnapKit](https://github.com/SnapKit/SnapKit)は、Auto Layoutベースのレイアウトを、swift DSLで快適に実装することが可能となるライブラリ。

コードベースでAuto Layoutによるレイアウト構築を行う場合、Layout Anchorsを用いるか、NSLayoutConstraintを用いるか、あるいはVisual Format Languageで記述するか、といった手段を取ることになる[^autolayout]。

例えば、myViewをviewに対して以下のルールで配置したい場合：

- myViewの上端は、viewの上端から8ptの余白を置く
- myViewの左右両端は、viewの左右両端から16ptの余白を置く

Layout Anchorsでは以下のような実装になる。

```swift
myView.translatesAutoresizingMaskIntoConstraints = false
myView.topAnchor.constraint(equalTo: view.topAnchor, constant: 8).isActive = true
myView.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16).isActive = true
myView.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16).isActive = true
```

まずはじめに`translatesAutoresizingMaskIntoConstraints`をfalseにして、viewが持つautoresizing maskがAuto Layout constraintsに変換されないよう制限し、自前で実装するconstraintsとの干渉を防ぐ必要がある。

また、constraintを定義する際は、sourceとtargetのviewからそれぞれAnchorを取り出して関係性を示し、最後に`isActive`をtrueにしてconstraintを有効にしなければならない。

constraintの都度`isActive`をtrueにするのが面倒であれば、以下のように`NSLayoutConstraint.activate([NSLayoutConstraint])`を用いれば良いが、それでも野暮ったい印象を受ける。

```swift
myView.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
  myView.topAnchor.constraint(equalTo: view.topAnchor, constant: 8),
  myView.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
  myView.trailing.constraint(equalTo: view.trailingAnchor, constant: -16)
])
```

これをSnapKitによって実装した場合、以下のようになる。

```swift
myView.snp.makeConstraints { make in
  make.top.equalToSuperview().offset(8)
  make.leading.equalToSuperview().offset(16)
  make.trailing.equalToSuperview().offset(-16)
}
```

まず、`translatesAutoresizingMaskIntoConstraints = false`を自前で実装する必要がなくなり、これの実装漏れに気付かずconstraintsが効かない…といった穴にハマることがなくなる。

また、`.makeConstraints`のclosureでviewに対する複数のconstraintをまとめて定義できることから、すっきり記述でき可読性が向上する。

加えて、`.equalTo`や`.lessThanOrEqualTo`、`.greaterThanOrEqualTo`メソッドが用意されており、これらを用いることでより自然な言葉でconstraintsを実装できる。

```swift
NSLayoutConstraint.activate([
	myView.topAnchor.constraint(equalTo: view.topAnchor, constant: 8)
])

myView.snp.makeConstraints { make in
	make.top.equalToSuperview().offset(8)
}
```

SnapKitの[ドキュメント](http://snapkit.io/docs/)を読めば、すぐに既存のAuto Layoutコードを置き換えられるはず。

いくつかサンプルコードを記載しておく。

```swift
// myViewとviewの端を一致させる
myView.snp.makeConstraints { make in
	make.edges.equalTo(view)
}

// myViewとviewの端を関連付け、上下8pt、左右16ptの余白を設ける
myView.snp.makeConstraints { make in
	make.edges.equalTo(view).inset(UIEdgeInsets(top: 8, left: 16, bottom: 8, right: 16))
}

// myViewの上端とviewの下端、myViewとviewの左右両端を関連付け、上に8pt、左右16ptの余白を設ける
myView.snp.makeConstraints { make in
	make.top.equalTo(view.snp.bottom).offset(8)
	make.leading.equalTo(view).offset(16)
	make.trailing.equalTo(view).offset(-16)
}

// myViewの上端とviewの下端、myViewとviewの左右両端を関連付け、上に8pt、左右16ptの余白を設ける。また、myViewの高さを4ptにする
myView.snp.makeConstraints { make in
	make.top.equalTo(view.snp.bottom).offset(8)
	make.leading.equalTo(view).offset(16)
	make.trailing.equalTo(view).offset(-16)
	make.height.equalTo(4)
}

// myViewの上端とviewの下端を関連付け、上に16ptの余白を設ける。また、myViewをviewの左右中央に配置する
myView.snp.makeConstraints { make in
	make.top.equalTo(view.snp.bottom).offset(16)
	make.centerX.equalTo(view)
}

// myViewの上下端をSafe Areaに沿わせる
myView.snp.makeConstraints { make in
	make.top.bottom.equalTo(view.safeAreaLayoutGuide)
}

// constraintごとにラベルを付与して、`Unable to simultaneously satisfy constraints.`ログを調べやすくする
myView.snp.makeConstraints { make in
	make.top.equalTo(view.snp.bottom).offset(8).labeled("myViewTopConstraint")
	make.leading.equalTo(view).offset(16).labeled("myViewLeadingConstraint")
	make.trailing.equalTo(view).offset(-16).labeled("myViewTrailingConstraint")
}
```

[^autolayout]: [Auto Layout Guide: Programmatically Creating Constraints](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/ProgrammaticallyCreatingConstraints.html)

