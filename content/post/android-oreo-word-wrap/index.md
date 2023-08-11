---
title: "Android 8.0.0で、不正な位置でテキストが折り返される"
date: 2018-11-10T23:00:00+09:00
description: "Android 8.0.0でのみ、不正な位置でテキストが折り返される現象に遭遇したので、メモ。"
slug: android-oreo-word-wrap
tags:
  - Android
---

[こちら](https://issuetracker.google.com/issues/64418117)のチケットで報告されているように、Android Oreo 8.0.0のTextViewにはテキスト折り返しのバグが存在する。

文節の長い文章をTextViewを表示させた場合、TextViewを画面幅一杯に広げている場合でも以下のように8.0.0のみ右端に大きな余白が空いてしまう場合がある。

|7.0|8.0.0|8.1.0|
|:---:|:---:|:---:|
|![7.0](20181206155954.webp)|![8.0.0](20181206160027.webp)|![8.1.0](20181206160042.webp)|

テキスト折り返しのルールは、Android 6.0で追加された[breakStrategy](https://developer.android.com/reference/android/widget/TextView.html#attr_android:breakStrategy)プロパティによって制御できる。このルールは以下の三種類あり、デフォルトでは最も手厚い`BREAK_STRATEGY_HIGH_QUALITY`が利用される。

- [BREAK_STRATEGY_SIMPLE](https://developer.android.com/reference/android/text/Layout.html#BREAK_STRATEGY_SIMPLE)
- [BREAK_STRATEGY_BALANCED](https://developer.android.com/reference/android/text/Layout.html#BREAK_STRATEGY_BALANCED)
- [BREAK_STRATEGY_HIGH_QUALITY](https://developer.android.com/reference/android/text/Layout.html#BREAK_STRATEGY_HIGH_QUALITY)

このうちデフォルトで用いられている`BREAK_STRATEGY_HIGH_QUALITY`によるテキスト折り返しについて8.0.0のみ実装にバグがあるようで、8.1.0ではこの点が修正されている。

8.0.0で発生するこのバグを迂回するためには、`BREAK_STRATEGY_SIMPLE`を明示的に指定する。

|8.0.0 HIGH_QUALITY|8.0.0 SIMPLE|8.1.0 HIGH_QUALITY|
|:---:|:---:|:---:|
|![8.0.0 HIGH_QUALITY](20181206160146.webp)|![8.0.0 SIMPLE](20181206160220.webp)|![8.1.0 HIGH_QUALITY](20181206160237.webp)|

