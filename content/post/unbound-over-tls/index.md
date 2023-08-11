---
title: "Unboundで構築したDNSフォワーダをDNS over TLSに対応させ、プライベートDNSとして利用可能にする"
date: 2022-12-04T16:21:23+09:00
description: "Unboundにover TLSを組み込み、Androidの「プライベートDNS」として利用可能にする。"
slug: unbound-over-tls
---

Androidスマートフォンのネットワーク設定をみていたら、「プライベートDNS」なる設定項目を見つけた。調べたところ、モバイル回線で用いるDNSを設定できるらしい。

UnboundでDNSフォワーダを構築し、これの `local-zone` で広告配信系ドメインをNXDOMAINとすることで、Webブラウジング中に表示される広告をブロックする方法について、過去に以下で書いた。

[Raspberry PiとUnboundによるDNSフォワーダの構築](https://blog.arkenous.net/raspberrypi-unbound/)

「プライベートDNS」として設定するためには、UnboundにDNS over TLSの機能を持たせることが求められるらしい。本記事では、UnboundにDNS over TLSの機能を追加し、モバイル回線に接続されたAndroidスマートフォンであってもDNSベースの広告ブロッキングを活用できる環境の構築方法を記載する。

##  前提

モバイル回線からDNSサーバに接続できるよう、DNSサーバがインターネット越しに見えている必要がある。VPSを借りるなりして、あらかじめサーバを構築しておく。

また、 `over TLS` とあるように、設定にあたってはドメイン名が必要となる。適当なドメインを持っていなければ、Google DomainsやValue Domainなどで調達し、サーバのグローバルIPアドレスとドメイン名をAレコードなどで紐付けておくこと。

本記事では、Arch LinuxでVPS上に構築した `dns.example.com` というドメイン名を割り当てたUnboundベースのDNSフォワーダに対し、over TLSの機能を付け加える。

## SSL/TLS証明書の取得

TLSで必要となるSSL/TLS証明書を取得する。この記事では、Let's Encryptによる証明書を用いる。

Let's Encryptによる証明書の取得には、 `certbot` を用いる。certbotを導入したら、以下コマンドでドメインに紐づく証明書を得る。

```
# certbot certonly --no-eff-email --agree-tos --email {証明書に載せる、連絡先E-mail} --standalone --non-interactive --domain {サーバのドメイン名}
```

上記では、スタンドアローンモードにより、ドメインの認証を通している。あらかじめ、サーバの80番ポートを開けておくこと。

証明書を無事取得できたら、certbotによる証明書の自動更新もあわせて組み込んでおくと良い。Arch Linuxの場合はsystemdベースの `certbot-renew.service` ならびに `certbot-renew.timer` が用意されており、以下コマンドでタイマーを有効にしておけば自動更新してくれる。

```
# systemctl start certbot-renew.timer
# systemctl enable certbot-renew.timer
```

また、証明書の自動更新時に、Unboundのリロードも含めておくと良い。 `certbot-renew.service` を編集し、certbotの `--post-hook` 機能を用いて実現する。

```
[Unit]
...

[Service]
...
ExecStart=/usr/bin/certbot -q renew --post-hook "systemctl reload unbound.service"
...
```

## Unboundへのover TLS機能の組み込み

SSL/TLS証明書を取得できたら、Unboundの設定を変更し、over TLS機能を組み込む。具体的には、 `unbound.conf` を以下のように編集する。

```conf
server:
    interface: 0.0.0.0@53 # IPv4のDNSポート
    interface: ::0@53 # IPv6のDNSポート
    interface: 0.0.0.0@853 # IPv4のDNS over TLSポート
    interface: ::0@853 # IPv6のDNS over TLSポート

    tls-service-key: "/path/to/letsencrypt/live/dns.example.com/privkey.pem"
    tls-service-pem: "/path/to/letsencrypt/live/dns.example.com/fullchain.pem"
    tls-port: 853
```

over TLS機能の組み込みにあたり必要となるのは、このあたり。 `access-control: 0.0.0.0/0 allow` としてアクセス可能にするのを忘れずに。また、 `hide-` 系プロパティや `harden-` 系プロパティは、なるべく適切に設定しとくと良い。キャッシュ系プロパティは、よしなに。

以下を `unbound.conf` に追加しておくと、このUnboundサーバからフォワードする際に、over TLSでやりとりしてくれる。

```conf
server:
    tls-system-cert: yes

forward-zone:
    name: "."
    forward-tls-upstream: yes

    # Google Public DNSを用いる場合
    forward-addr: 8.8.8.8@853#dns.google
    forward-addr: 8.8.4.4@853#dns.google
    forward-addr: 2001:4860:4860::8888@853#dns.google
    forward-addr: 2001:4860:4860::8844@853#dns.google
```

`unbound.conf` の編集が済んだら、以下コマンドで設定ファイル中に明らかな記載ミスがないかを確認する。

```
# unbound-checkconf
```

ミスがないことが確認できたら、以下コマンドでUnboundを起動（あるいは再起動）する。

```
（まだ起動していないなら）
# systemctl start unbound

(起動済みなら）
# systemctl restart unbound
```

起動したら、以下でサービスの状態に問題がないかを確認しておく。

```
# systemctl status unbound.service
```

問題なさそうなら、実際にAndroidスマートフォンの「プライベートDNS」にて、構築したDNSサーバのドメイン名を入れてみる。モバイル回線に切り替えた状態で、Webブラウザで名前解決できるようなら、問題ない。

`unbound.conf` で以下のように設定してDNSクエリ・レスポンスログを出力させ

```conf
server:
    log-time-ascii: yes
    log-queries: yes
    log-replies: yes
    log-tag-queryreply: yes
```

以下でログを監視し

```
# journalctl -f -u unbound
```

名前解決されている状況を眺めてみるのも良い。

## 参考
- [Private DNS Server for Android 9](https://andreasch.com/2019/01/29/unbound-privdns/)
- [Unbound - ArchWiki](https://wiki.archlinux.org/title/Unbound)
- [Welcome to the Certbot documentation! — Certbot 2.0.0 documentation](https://eff-certbot.readthedocs.io/en/stable/)
- [Certbot - ArchWiki](https://wiki.archlinux.org/title/Certbot)

