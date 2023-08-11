---
title: "アスペクト比を求めるアルゴリズム，ユークリッドの互除法"
date: 2012-06-09T14:31:00+09:00
description: "アスペクト比を求めるためのユークリッドの互除法のアルゴリズムを、CとJavaで実装してみた。"
slug: euclid
---

アスペクト比を求めるためのユークリッドの互除法のアルゴリズムを，まずはC言語で実装してみた．

```c
#include <stdio.h>

// 与えられた2数から最大公約数を求める
int getGCD(int x, int y) {
 int tmp = 0;

 // xよりもyの値が大きければ，tmp変数を使って入れ替える
 if (x < y) {
	tmp = x;
	x = y;
	y = tmp;
 }

 // yが自然数でなければ-1を返す
 if (y <= 0) {
	return -1;
 }

 // xをyで割った余りが0であれば，それが最大公約数
 if (x % y == 0) {
	return y;
 }

 // 再帰関数（上のif文に引っかかるまでgetGCDを繰り返す）
 return getGCD(y, x % y);
}

int main() {
 int x, y, n;
 int aspX, aspY;

 x = 1920;
 y = 1080;

 if ((n = getGCD(x, y)) > 0) {
	printf("最大公約数は%d\n", n);

	aspX = x / n;
	aspY = y / n;

	printf("アスペクト比を表示します\n");
	printf("%d : %d\n", aspX, aspY);
 }
}
```

上のソースを参考に，AndroidにJavaでクラスを実装した．

```java
public class GetGCD {
  public static int getGCD(int x, int y) {
int tmp = 0;

// xよりもyの値が大きければ，tmp変数を使って入れ替える
if (x < y) {
  tmp = x;
  x = y;
  y = tmp;
}

// yが自然数でなければ-1を返す
if (y <= 0) {
  return -1;
}

// xをyで割った余りが0であれば，それが最大公約数
if (x % y == 0) {
  return y;
}

// 再帰関数（上のif文に引っかかるまでgetGCDを繰り返す）
return getGCD(y, x % y);
  }
}
```

呼び出し元で

```java
int gcd = GetGCD.getGCD(1920, 1080);
```

としてやると，`getGCD`に与えた2数（上の例では`1920`と`1080`）の最大公約数が`gcd`に返される．あとは与えた2数をこの最大公約数で割ればいい．

```java
String aspect = String.valueOf(1920 / gcd) + ":" + String.valueOf(1080 / gcd);
```

これで比が求められる（上記の例では，`aspect`に`16:9`という文字列が入る）．
