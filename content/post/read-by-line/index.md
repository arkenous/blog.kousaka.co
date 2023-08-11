---
title: "シェルスクリプトでの行ごとの読み込み"
date: 2012-06-23T14:35:00+09:00
description: "あるコマンドの出力をパイプで受け取って，行ごとに変数に放り込んで処理するシェルスクリプトを書いた。"
slug: read-by-line
---

あるコマンドの出力をパイプで受け取って，行ごとに変数に放り込んで処理するシェルスクリプトを書いた（output.sh）．

```sh
!/bin/sh
while read input;
do
  echo $input
done;
```

これは，次のように実行する．

```sh
echo "aaa\nbbb\nccc" | ./output.sh
```

実行した結果は以下のようになる．

```
aaa
bbb
ccc
```

参考：[1行ずつのファイル読み込み | hiro345](http://www.sssg.org/blogs/hiro345/archives/6559.html)
 
