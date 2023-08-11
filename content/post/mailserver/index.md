---
title: "Postfix+DovecotでSSLに対応し，SpamAssassinを組み込み，SPF+DKIM+DMARCを設定したメールサーバの構築"
date: 2016-04-16T14:00:00+09:00
description: "メール転送エージェントにPostfix，POP3/IMAPサーバにDovecotを用い，SSLによる通信経路の暗号化をし，スパムフィルタのSpamAssassinを組み込み，SPFやDKIM，DMARCを設定したメールサーバの構築手順について書く．"
slug: mailserver
---

メール転送エージェントにPostfix，POP3/IMAPサーバにDovecotを用い，SSLによる通信経路の暗号化をし，スパムフィルタのSpamAssassinを組み込み，SPFやDKIM，DMARCを設定したメールサーバの構築手順について書く．

ここでは，メールサーバのFQDNが`mail.example.net`，作成するメールアカウントを`test@example.net`と仮定して書いているため，自身の環境に合わせて読み替えてほしい．なお，Arch Linuxベースでの構築を想定している．

## ソフトウェアのインストール

まず，メールサーバ構築に必要なソフトウェアをインストールする．

```
# pacman -S postfix dovecot opendkim
```

インストールできたら，まずはDovecotの設定を行う．

## Dovecot

Dovecotの設定に入る前に，まずはクライアントとサーバ間でSSLでの暗号化通信を行うためのSSL証明書を取得する．今回は，[Let's Encrypt](https://letsencrypt.org)を利用して証明書を取得する．

### SSL証明書の取得

Let's Encryptでは，`certbot`というクライアントを用いることで簡単に証明書を取得できる．これは以下のコマンドでインストールする．

```
# pacman -S certbot
```

インストールできたら，以下のコマンドで証明書を取得する．

```
# certbot certonly
```

また，Let's Encryptで発行される証明書の有効期限は90日のみなので，cronなどを用いて期限前に証明書が自動更新されるように準備するのがベターである．
ここではcertbotの詳細な説明は省く．ArchWikiの[該当ページ](https://wiki.archlinux.org/index.php/Let%E2%80%99s_Encrypt)に詳細な説明があるため，より詳しく知りたい場合はそちらを参照してほしい．

### Dovecotの設定

SSL証明書が取得できたところで，次はDovecotの設定を行う．まずは，Dovecotに付属して用意されている設定サンプルをDovecotの設定ディレクトリである`/etc/dovecot/`以下にコピーする．

```
# cp /usr/share/doc/dovecot/example-config/dovecot.conf /etc/dovecot/
# cp -r /usr/share/doc/dovecot/example-config/conf.d /etc/dovecot/
```

コピーできたら，`/etc/dovecot/conf.d/10-mail.conf`を以下のDiffを参考に編集する．

```diff
# diff /usr/share/doc/dovecot/example-config/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf
30c30
< #mail_location =
---
> mail_location = maildir:~/Maildir
```

次に，`/etc/dovecot/conf.d/10-master.conf`を以下のDiffを参考に編集する．

```diff
# diff /usr/share/doc/dovecot/example-config/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf
96,98c96,100
<   #unix_listener /var/spool/postfix/private/auth {
<   #  mode = 0666
<   #}
---
>   unix_listener /var/spool/postfix/private/auth {
> mode = 0660
> user = postfix
> group = postfix
>   }
```

また，`/etc/dovecot/conf.d/10-ssl.conf`を以下のDiffを参考に編集する．

```diff
# diff /usr/share/doc/dovecot/example-config/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf
6c6
< #ssl = yes
---
> ssl = yes
12,13c12,13
< ssl_cert = </etc/ssl/certs/dovecot.pem
< ssl_key = </etc/ssl/private/dovecot.pem
---
> ssl_cert = </etc/letsencrypt/live/example.net/fullchain.pem
> ssl_key = </etc/letsencrypt/live/example.net/privkey.pem
```

加えて，お好みで`/etc/dovecot/conf.d/15-mailboxes.conf`を編集する．僕自身は以下のように変更した．

```
namespace inbox {
  mailbox Drafts {
auto = subscribe
special_use = \Drafts
  }
  mailbox Junk {
auto = subscribe
special_use = \Junk
  }
  mailbox Trash {
auto = subscribe
special_use = \Trash
  }
  mailbox Sent {
auto = subscribe
special_use = \Sent
  }
  mailbox Archive {
auto = subscribe
special_use = \Archive
  }
}
```

## Postfix

Postfixに関係する設定ファイルは`/etc/postfix/`の下に置かれている．まずは`main.cf`を以下のDiffを参考に編集する．

```diff
# diff /etc/postfix/main.cf.orig /etc/postfix/main.cf
94c94
< #myhostname = host.domain.tld
---
> myhostname = mail.example.net
102c102
< #mydomain = domain.tld
---
> mydomain = example.net
118c118
< #myorigin = $mydomain
---
> myorigin = $mydomain
132c132
< #inet_interfaces = all
---
> inet_interfaces = all
180c180
< #mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
---
> mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
403a404
> alias_maps = hash:/etc/postfix/aliases
413a415
> alias_database = $alias_maps
434c436
< #home_mailbox = Maildir/
---
> home_mailbox = Maildir/
568c570
< #smtpd_banner = $myhostname ESMTP $mail_name
---
> smtpd_banner = $myhostname ESMTP unknown
678a681,692
>
> smtpd_sasl_type = dovecot
> smtpd_sasl_path = private/auth
> smtpd_sasl_auth_enable = yes
>
> smtpd_use_tls = yes
> smtpd_tls_cert_file = /etc/letsencrypt/live/example.net/fullchain.pem
> smtpd_tls_key_file = /etc/letsencrypt/live/example.net/privkey.pem
> smtpd_tls_session_cache_database = btree:/var/lib/postfix/smtpd_scache
>
> disable_vrfy_command = yes
> smtpd_helo_required = yes
```

次に，`/etc/postfix/master.cf`を以下のDiffを参考に編集する．

```diff
# diff master.cf.proto master.cf
17c17
< #submission inet n   -   n   -   -   smtpd
---
> submission inet n   -   n   -   -   smtpd
19,20c19,20
< #  -o smtpd_tls_security_level=encrypt
< #  -o smtpd_sasl_auth_enable=yes
---
>   -o smtpd_tls_security_level=encrypt
>   -o smtpd_sasl_auth_enable=yes
28c28
< #smtps inet  n   -   n   -   -   smtpd
---
> smtps inet  n   -   n   -   -   smtpd
30,31c30,31
< #  -o smtpd_tls_wrappermode=yes
< #  -o smtpd_sasl_auth_enable=yes
---
>   -o smtpd_tls_wrappermode=yes
>   -o smtpd_sasl_auth_enable=yes
```

また，rootアカウント宛のメールを受信できるように，`/etc/postfix/aliases`を以下のDiffを参考に編集する．

```diff
# diff aliases.origin aliases
12c12
< #root:		you
---
> root:		test
```

`/etc/postfix/aliases`ファイルを編集できたら，以下のコマンドを実行する．

```
# newaliases
```

最後に，`/etc/services`にsmptsのポート番号を追加する．以下の文字列を追加すればいい．

```
smtps 465/tcp
smtps 465/udp
```

## SPFの設定

### 送信側設定

`example.net`ドメインを管理しているDNSのゾーン情報に，以下のようなTXTレコードを追加する．

```dns
@ IN TXT "v=spf1 +a:example.net +ip4:MAIL SERVER'S IP ADDRESS HERE ~all"
```

### 受信側設定（検証）

まず，検証に必要となる`python-postfix-policyd-spf`をAURから以下のコマンドでインストールする．

```
$ pacaur -S python-postfix-policyd-spf
```

上記コマンドを実行し，依存パッケージの`python-pyspf`のインストール中に`python-setuptools`で`~ exists in file system`というようなコンフリクトエラーが出た場合は，以下のコマンドを実行する．

```
# pacman -S --force python-setuptools
```

インストールできたら，コメントの付いた設定ファイルである`/etc/python-policyd-spf/policyd-spf.conf.commented`を参考に`/etc/python-policyd-spf/policyd-spf.conf`を編集する．

僕自身は以下のようにデフォルトのままで運用している．

```
debugLevel = 1
defaultSeedOnly = 1

HELO_reject = SPF_Not_Pass
Mail_From_reject = Fail

PermError_reject = False
TempError_Defer = False

skip_addresses = 127.0.0.0/8,::ffff:127.0.0.0/104,::1
```

設定ファイルを編集できたらPostfixの設定ファイルを編集し，このSPFバリデータをPostfixから扱えるようにする．まずは`/etc/postfix/master.cf`に以下を追加する．

```
policy-spf  unix-nn-0spawn
   user=nobody  argv=/usr/bin/policyd-spf
```

次に，`/etc/postfix/main.cf`に以下を追加する．

```
policy-spf_time_limit = 3600s
smtpd_recipient_restrictions=
 permit_sasl_authenticated
 permit_mynetworks
 reject_unauth_destination
 check_policy_service unix:private/policy-spf
```

## DKIMの設定

### 送信側設定

まずは`/usr/share/doc/opendkim/opendkim.conf.sample`を`/etc/opendkim/opendkim.conf`にコピーする．

```
# cp /usr/share/doc/opendkim/opendkim.conf.sample /etc/opendkim/opendkim.conf
```

コピーできたら，`/etc/opendkim/opendkim.conf`を以下のDiffを参考に編集する．

```diff
# diff opendkim.conf.sample opendkim.conf
111c111
< # BaseDirectory		/var/run/opendkim
---
> BaseDirectory		/var/lib/opendkim
162c162
< Domain			example.com
---
> Domain			example.net
203c203
< # ExternalIgnoreList	filename
---
> ExternalIgnoreList	refile:/etc/opendkim/TrustedHosts
229c229
< # InternalHosts		dataset
---
> InternalHosts		refile:/etc/opendkim/TrustedHosts
247c247
< KeyFile			/var/db/dkim/example.private
---
> KeyFile			/etc/opendkim/keys/example.net/20160705.private
258c258
< # KeyTable		dataset
---
> KeyTable		refile:/etc/opendkim/KeyTable
339c339
< # Mode			sv
---
> Mode			sv
572c572
< Selector		my-selector-name
---
> Selector		20160705
632c632
< # SigningTable		filename
---
> SigningTable		refile:/etc/opendkim/SigningTable
660c660
< Socket			inet:port@localhost
---
> Socket			inet:8891@localhost
727c727
< # TemporaryDirectory	/tmp
---
> TemporaryDirectory	/run/opendkim
753c753
< # UMask			022
---
> UMask			022
763c763
< # UserID		userid
---
> UserID		opendkim
```

なお，設定ファイル中で指定している`Selector`，ここでは`20160705`としているが，これは自由な文字列を設定してよい．

編集できたら，まずは以下のコマンドを実行する．

```
# mkdir /var/lib/opendkim && chown -R opendkim:mail /var/lib/opendkim
```

次に，`/etc/opendkim/KeyTable`を作成し，以下の文字列を追加する．

```
20160705._domainkey.example.net example.net:20160705:/etc/opendkim/keys/example.net/20160705.private
```

さらに，`/etc/opendkim/SigningTable`を作成し，以下の文字列を追加する．

```
*@example.net   20160705._domainkey.example.net
```

最後に，`/etc/opendkim/TrustedHosts`を作成し，以下の文字列を追加する．

```
127.0.0.1
```

以上の設定ができたら，以下のコマンドでDKIMの一時ディレクトリを作成し，権限を設定する．

```
# mkdir /run/opendkim
# chown -R opendkim:mail /run/opendkim
```

次に，DKIM署名用の秘密鍵・公開鍵の作成をする．まず，これらファイルを保存するディレクトリを以下のコマンドで作成する．

```
# mkdir /etc/opendkim/keys/example.net
```

ディレクトリを作成できたら，以下のコマンドで鍵ファイルを作成する．

```
# opendkim-genkey -D /etc/opendkim/keys/example.net -d example.net -s 20160705
```

鍵ファイルを作成できたら，DKIM設定ディレクトリの権限を以下のコマンドで変更する．

```
# chown -R opendkim:mail /etc/opendkim
```

権限を変更できたら，DKIMの公開鍵レコードとADSPレコードの二つを`example.net`ドメインを管理しているDNSのゾーン情報に追加する．

まず，公開鍵レコードについては以下のものを追加する．

```dns
20160705._domainkey.example.net. IN TXT "v=DKIM1; k=rsa; p=<公開鍵データ>"
```

なお，`<公開鍵データ>`には，公開鍵ファイル`/etc/opendkim/keys/example.net/20160705.txt`の`p=`以降に記載されている文字列をそのまま写せばいい．

```dns
20160705._domainkey	IN	TXT	( "v=DKIM1; k=rsa; "
	"p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDPtom+SH5X2+EHL4eBzqRGxoqBN61GOpD0tv1f2juNqVbzvjzDBfRfhwJjE0AmF0L8YrpkqZJySUVKNSpVmt7NLBTUY8WiYe+VT+iBc+bbDtYOmOTAszIiclJ4EZcq4xRm2M/ZSoipGNEp4722EIBGT0lmFFmGv059wiSYBcTD4QIDAQAB" )  ; ----- DKIM key 20160705 for example.net
```

次に，ADSPレコードは以下のものを追加する．

```dns
_adsp._domainkey.example.net. IN TXT "dkim=unknown"
```

### 受信側設定（検証）

`/etc/postfix/main.cf`に以下を追加し，PostfixからDKIMの検証を行えるようにする．

```
non_smtpd_milters = inet:127.0.0.1:8891
smtpd_milters = inet:127.0.0.1:8891
milter_default_action = accept
```

## DMARCの設定

DMARCとは，上で設定したSPFやDKIMといった認証技術を利用し，詐称された（なりすまされた）メールを受信者がどう扱うべきかをドメイン管理者が宣言・制御するものである．

### 送信側設定

送信側でこれを利用するためには，SPFやDKIMの設定と同様にこのドメインを管理しているDNSのゾーン情報にTXTレコードを一つ追加するだけでいい．

具体的には，以下のようなTXTレコードを追加する．

```dns
_dmarc.example.net. IN TXT "v=DMARC1; p=quarantine; pct=100; rua=mailto:admin@example.net; sp=reject;"
```

レコードのエントリ名は，上記例のようにドメイン名の前に`_dmarc.`を追加する．レコードの中で必須のものは，DMARCのバージョンを指定する`v`タグと，SPFやDKIMによる認証が失敗した場合に受信者にどう処理して欲しいかのポリシーを宣言する`p`タグの二つである．DMARCのバージョン指定については，現在はバージョン1しか無いようなので`v=DMARC1;`と指定すればいい．ポリシーの宣言については，以下に挙げる3種類から選ぶ．

- `none` : 何もしない．該当メールをレポートに書くだけ
- `quarantine` : 該当メールに迷惑メールマークを付ける
- `reject` : SMTP層でメールを拒否させる

`pct`タグは任意で指定できるもので，該当したメールのうちどれだけの割合のメールを実際にポリシーに従って処理するかを指定する．`pct`タグを指定しなかった場合はDMARCのチェックを通過しなかった全てのメールについてポリシーに従って処理される．DMARCによる認証結果は日々レポートとして記録されるが，これを特定のメールアドレスで受信したい場合は，`rua`タグで指定する．`sp`タグでは，ドメインのサブドメインを用いたメールについてどう処理するかを指定する．指定する内容は`p`タグと同様である．

DMARCを設定する際には，徐々にルールを厳しくしていく方法が望ましいとされている．まずは`p`タグで`none`を指定して`rua`タグで指定したメールアドレスに届く日々のレポートで結果を確認し，問題が無ければ`p`タグで`quarantine`を指定する．そして全てのメールで意図したように署名がなされ，問題なく運用できていることが確認できたら`p`タグに`reject`を指定する．また，`quarantine`や`reject`を指定する際に`pct`タグで初めは`20`などの低い割合を指定し，徐々に`100`に近づけていく方法を取ることでよりじっくりとDMARCを導入できる．

### 受信側設定（検証）

まず，以下のコマンドで検証に必要となるパッケージをインストールする．

```
# pacman -S opendmarc
```

インストールできたら，サンプルの設定ファイルを以下のコマンドでコピーする．

```
# cp /etc/opendmarc/opendmarc.conf.sample /etc/opendmarc/opendmarc.conf
```

コピーできたら，`/etc/opendmarc/opendmarc.conf`の該当箇所を以下のように編集する．

```diff
# diff /etc/opendmarc/opendmarc.conf.sample /etc/opendmarc/opendmarc.conf
276c276
< # Socket inet:8893@localhost
---
> Socket  unix:/run/opendmarc/dmarc.sock
346c346
< # UMask 077
---
> UMask 007
```

また，DMARCの実行ユーザを指定し，`/run/`以下に自動的にソケットファイル用のディレクトリを作成させる（ランタイムディレクトリ）ために，`/usr/lib/systemd/system/opendmarc.service`の`[Service]`内に以下を追加する．

```
User=opendmarc
Group=postfix
RuntimeDirectory=opendmarc
```

PostfixでDMARCの検証ができるように，`/etc/postfix/main.cf`に以下を追加する．

```
non_smtpd_milters = inet:127.0.0.1:8891, unix:/run/opendmarc/dmarc.sock
smtpd_milters = inet:127.0.0.1:8891, unix:/run/opendmarc/dmarc.sock
```

なお，DKIMの受信側設定をした場合は，該当箇所のその後ろに`unix:/run/opendmarc/dmarc.sock`を付け足せばいい．ただし，DKIM milter（`inet:127.0.0.1:8891`）の前にDMARC milter（`unix:/run/opendmarc/dmarc.sock`）を入れないこと．

## SpamAssassinの導入

スパムフィルタリングのプログラムであるSpamAssassinをPostfixに組み込む．まずは以下のコマンドを実行して必要なパッケージをインストールする．

```
# pacman -s spamassassin
```

インストールできたら，SpamAssassinの設定ファイルである`/etc/mail/spamassassin/local.cf`を適宜編集する．ただ，基本的にはデフォルトのままでも使えるようだ．

次に，SpamAssassinのルールを以下のコマンドで更新する．

```
# sa-update
# sa-compile
```

また，Systemdのタイマーを用いてルールを自動的・定期的に更新できるように設定する．まずは`/etc/systemd/system/spamassassin-update.service`を以下の内容で作成する．

```
[Unit]
Description=spamassassin housekeeping stuff

[Service]
User=spamd
Group=spamd
Type=oneshot
ExecStart=/usr/bin/vendor_perl/sa-update --allowplugins
ExecStart=/usr/bin/vendor_perl/sa-compile
# You can automatically train SA's bayes filter by uncommenting this line and specifying the path to a mailbox where you store email that is spam (for ex this could be yours or your users manually reported spam)
#ExecStart=/usr/bin/vendor_perl/sa-learn --spam <path to your spam>
```

最後の`ExecStart`について，特定のメールディレクトリで既にスパムメールを溜めている場合は先頭の`#`を消して`--spam`オプションでディレクトリのパスを指定することで，迷惑メールのフィルタを学習させられる．[^learning]

そして，上記サービスを毎日実行するためのタイマーを`/etc/systemd/system/spamassassin-update.timer`というファイル名で以下の内容で作成する．

```
[Unit]
Description=spamassassin house keeping

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

作成したサービスファイルをSpamAssassinのサービスファイルから読み込むために，まずは以下のコマンドを実行する．

```
# cp /usr/lib/systemd/system/spamassassin.service /etc/systemd/system/
```

そして，`/etc/systemd/system/spamassassin.service`の`[Unit]`内に以下を追記する．

```
PartOf=spamassassin-update.service
```

この設定により，タイマーが起動する前にSpamAssassinのspamdが再起動される．

さて，タイマーを起動する前に，SpamAssassinのルール更新に関係するディレクトリをspamdユーザが書き込めるように，所有権を変更する．具体的には以下のコマンドを実行する．

```
# chown -R spamd:spamd /etc/mail/spamassassin/sa-update-keys
# chown -R spamd:spamd /var/lib/spamassassin
```

以上が完了したら，以下のコマンドでSpamAssassinルールの自動更新タイマーを起動し，自動起動するように設定する．

```
# systemctl start spamassassin-update.timer
# systemctl enable spamassassin-update.timer
```

タイマーの設定ができたら，次はPostfixからSpamAssassinを扱えるように，`/etc/postfix/master.cf`を編集する．まずはメール受信・送信に用いているプロトコルの下に以下を追記する．

```
-o content_filter=spamassassin
```

例えばsmtpを用いている場合は，以下のようになる．

```
smtpinet  n  -  n  -  -  smtpd
 -o content_filter=spamassassin
```

smtpsやsubmissionの場合も同じように設定する．複数プロトコルに設定してももちろん大丈夫，なはず．

また，以下も`/etc/postfix/master.cf`の末尾に追加する．

```
spamassassinunix  -  n  n  -  -  pipe
 flags=R user=spamd argv=/usr/bin/vendor_perl/spamc -e /usr/bin/sendmail -oi -f ${sender} ${recipient}
```

## メールユーザの追加

メールユーザを追加する前に，ユーザ作成時にMaildirが自動生成されるように設定する．
具体的には，以下のコマンドを実行すればいい．

```
# mkdir /etc/skel/Maildir
# mkdir /etc/skel/Maildir/{new,temp,cur}
# chmod -R 700 /etc/skel/Maildir
```

実行したら，新規メールユーザを作成する．以下のコマンドで，`test`というユーザを作成している．

```
# useradd -m -s /sbin/nologin test
# passwd test
# saslpasswd2 -u example.net -c test
# chgrp postfix /etc/sasldb2
```

`passwd`や`saslpasswd2`の実行時にパスワード入力を促されると思うので，あまり弱くないパスワードを入力すること．これがメールサーバへログインするためのパスワードとなる．

## MXレコードの追加

さて，サービスを起動する前に`example.net`ドメインを管理しているDNSのゾーン情報にMXレコードを追加しなければならない．
具体的には，以下のものを追加すればいい．

```dns
@ IN MX 10 example.net.
```

## サービスの起動及び，自動起動の設定

以下のコマンドを実行して，今まで設定してきたサービスの起動及びサーバ起動時に自動起動する設定をする．

```
# systemctl start opendkim.service
# systemctl enable opendkim.service
# systemctl start opendmarc.service
# systemctl enable opendmarc.service
# systemctl start spamassassin.service
# systemctl enable spamassassin.service
# systemctl start postfix.service
# systemctl enable postfix.service
# systemctl start dovecot.service
# systemctl enable dovecot.service
```

## 動作テスト

現時点でメールサーバが稼働している．Thunderbird等のメールクライアントを用いてメールサーバにログインして他のアドレスに対してメールを送信できるか，また受信できるかを確認してほしい．

## オープンリレー確認

最後に，頑張って構築したメールサーバが第三者によって中継利用されないか，踏み台にされないかを必ず確認してほしい．
例えば[第三者中継チェック RBL.JP](http://www.rbl.jp/svcheck.php)を用い，`ホスト名`にメールアドレスの@以下，ここで説明した例では`example.net`を入力して`Check`ボタンを押すと自動的に確認をしてくれる．全てのテストにパスして青い文字で`no relays accepted`と出ればひとまず大丈夫なはずだ．

---

### 参考

- [ArchLinux上にDovecot+Postfixでメールサーバを構築 - せかいろぐ](http://sekai.hatenablog.jp/entry/2013/05/11/193700)
- [Postfix SPF 認証 設定メモ（postfix-policyd-spf-python） | あぱーブログ](https://blog.apar.jp/linux/766/)
- [Postfix DKIM 認証 設定メモ（CentOS6.5＋OpenDKIM） | あぱーブログ](https://blog.apar.jp/linux/856/)
- [python-postfix-policyd-spfを使う - スコトプリゴニエフスク通信](http://perezvon.hatenablog.com/entry/20080205/1202199797)

---

[^learning]: 僕自身は，Maildirの権限の関係からrootユーザのcrontabに以下を追加し，ベイズフィルタの学習を毎日実行している．`0 1 * * * /usr/bin/vendor_perl/sa-learn -u spamd --dbpath /var/lib/spamassassin/.spamassassin/ --spam /home/*/Maildir/.Spam/cur >/dev/null 2>&1 ; /usr/bin/vendor_perl/sa-learn -u spamd --dbpath /var/lib/spamassassin/.spamassassin/ --ham /home/*/Maildir/.Archive/cur >/dev/null 2>&1`

