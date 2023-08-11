---
title: "Postfixメールサーバを冗長化した"
date: 2015-10-23T15:00:00+09:00
description: "メールサーバを冗長構成にせず一つだけしか動かしていない場合，そのサーバに何からの異常が発生して正常に動作しなくなった場合にメールの受信や送信が出来なくなる可能性がある．そこで，メールサーバの冗長化を行うことでこの問題を回避する．"
slug: redundant-postfix
---

メールサーバを冗長構成にせず一つだけしか動かしていない場合，そのサーバに何からの異常が発生して正常に動作しなくなった場合にメールの受信や送信が出来なくなる可能性がある．そこで，メールサーバの冗長化を行うことでこの問題を回避する．

まずはセカンダリサーバにPostfixを以下のコマンドでインストールする．

```
$ yaourt -S postfix
```

インストールできたら設定を行っていく．まずは`/etc/postfix/main.cf`を次のように編集する．

```
myhostname = secondary.hoge.net
mydomain = hoge.net
myorigin = $mydomain

inet_interfaces = all

mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

relay_domains = hoge.net

transport_maps = hash:/etc/postfix/transport
alias_maps = hash:/etc/postfix/aliases
alias_database = $alias_maps
smtpd_banner = $myhostname ESMTP unknown

disable_vrfy_command = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/relay-password
smtp_sasl_security_options = noanonymous
smtpd_helo_required = yes
smtp_tls_security_level = may
```

次に，`/etc/postfix/transport`の設定を行う．`/etc/postfix/transport`に以下の行を追記する．

```
hoge.net  smtp:[primary.hoge.net]:587
```

なお，ホスト名を`[]`で囲っているのは，転送する際にDNSのMXレコードを用いた検索を行わせないため．

編集できたら，以下のコマンドを実行する．

```
# postmap /etc/postfix/transport
```

次に，`/etc/postfix/aliases`の設定を行う．`/etc/postfix/aliases`に以下の行を追記する．

```
user: user@hoge.net
```

`user`はプライマリのメールサーバにおいて設定したメールアカウントのユーザ名を記述する．

編集できたら，以下のコマンドを順に実行する．

```
# postalias /etc/postfix/aliases
# newaliases
```

次に，プライマリのメールサーバにおける`user`メールアカウントにアクセスするためのパスワード設定ファイルを作成する．これは，プライマリのメールサーバにおけるメールアカウントにSMTP Authを設定しているため必要だった．

`/etc/postfix/relay-password`をroot権限で作成し，以下のように編集した．

```
[primary.hoge.net]:587 user:Input user's password
```

編集できたら，このファイルに対してroot以外のユーザが閲覧および編集を行えないように，以下のコマンドで権限を編集する．

```
# chmod 600 /etc/postfix/relay-password
```

また，このファイルをPostfixが扱えるように，以下のコマンドを実行する．

```
# postmap relay-password
```

以上でPostfixの設定は完了した．

ファイアウォールにおいて，Postfixに関係する465番ポートや587番ポートを開放しておく．また，DNSにおいてMXレコードにsecondary.hoge.netを追加し，プライマリよりも大きい優先度，例えばプライマリが10であれば100のように設定する．

これらが終われば，あとは以下のコマンドでPostfixの起動と，システム起動時に自動起動されるようにする．

```
# systemctl start postfix
# systemctl enable postfix
```

これで冗長化が完了した．

プライマリサーバが何らかの理由で正常に動作しなくなっても送信されたメールはセカンダリサーバにプールされ，5日間はプライマリサーバへの再送を自動的に行ってくれるはずだ．なので，プライマリサーバが落ちてから5日経過するまでに何とかすればいい．

なお，この猶予時間は別途設定が可能なので，興味のある人は調べて欲しい．

