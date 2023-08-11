---
title: "GnuPGの環境を新サーバに移行したログ"
date: 2014-10-03T06:00:00+09:00
slug: migrating-gnupg
---

以下のコマンドを実行する

```sh
$ mkdir ~/.gnupg
$ chmod 700 ~/.gnupg
$ cp /usr/share/gnupg/gpg-conf.skel ~/.gnupg/gpg.conf
```

`~/.gnupg/gpg.conf`に以下の行を追記する．

```conf
display-charset utf-8
personal-digest-preferences SHA512
cert-digest-algo SHA512
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
personal-cipher-preferences TWOFISH CAMELLIA256 AES 3DES
use-agent
no-permission-warning
lock-never
```

`~/.gnupg/gpg-agent.conf`を作成し，以下の行を追記する．

```conf
pinentry-program /usr/bin/pinentry-curses
```

`~/.gnupg/`に作成したファイルのパーミッションを以下のコマンドで変更する．

```sh
$ chmod 600 ~/.gnupg/gpg-agent.conf
$ chmod 600 ~/.gnupg/gpg.conf
```

`/etc/pacman.conf`の以下の行をコメントアウトする

```conf
    GPGDir = /etc/pacman.d/gnupg/
```

旧サーバの鍵ファイルを以下のコマンドでエクスポートする．

```sh
$ gpg -o pub.key --export hoge@foo.bar
$ gpg -o sec.key --export-secret-key hoge@foo.bar
$ gpg --export-ownertrust > hogetrust
```

エクスポートできたら，scpなどの盗聴されない方法でこれらを新サーバに転送する．
次に，新サーバにおいて以下のコマンドでこれらをインポートする．

```sh
$ gpg --import pub.key
$ gpg --import --allow-secret-key-import sec.key
$ gpg --import-ownertrust hogetrust
```

たぶんこれで大丈夫なはず…

