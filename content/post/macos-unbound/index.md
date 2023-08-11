---
title: "macOSローカルにUnboundでDNSフォワーダを構築する"
date: 2022-06-04T15:50:12+09:00
description: "macOSローカルにDNSキャッシュサーバを構築し、名前解決をローカルで完結させ、Webブラウジング等を快適にする。"
slug: macos-unbound
---

**割と特殊な内容かなと思うので、やるなら自己責任でお願いします…。**

[LAN内部にUnboundサーバを構築する記事は書いた](https://blog.arkenous.net/raspberrypi-unbound/)が、クライアント内部にも同様にUnboundサーバを抱えさせれば、より高速に名前解決できるかなと思い、macOS上にUnboundを構築してみた。（つまるところ、UnboundによるDNSスタブリゾルバの構築）

とはいえ、対応自体は以下記事に沿って進めれば良い。個人的なメモとして、本記事を残しておく。

[Unbound DNS server on macOS - sizeof(cat)](https://sizeof.cat/post/unbound-on-macos/)

まずは、Homebrewでunboundパッケージを入れる。

```
$ brew install unbound
```

DNSSEC検証に用いるroot keyを得る。ただ、DNSフォワーダとかDNSスタブリゾルバとして、自身で反復問い合わせしないのであれば、多分要らない…。

```
# /usr/local/sbin/unbound-anchor -a /usr/local/etc/unbound/root.key
```

Unbound controlで必要となる証明書を作る。

```
# /usr/local/sbin/unbound-control-setup -d /usr/local/etc/unbound
```

`/usr/local/etc/unbound/unbound.conf` を自分好みに編集して、Unboundを設定する。僕の環境では、LAN内部に別のUnboundサーバを構築したので、キャッシュにヒットしなかったらそちらにフォワードさせる形にした。

なお、このファイルはUnboundに付属するサンプルファイルをベースに、usernameがHomebrewのインストール操作を実行したユーザ自身に置き換えられている。設定ファイル中で該当箇所がコメントアウトされているため、先頭のシャープを消しておくと良い。

以下の例では、変更した内容に絞って記載した。

```conf
server:
	verbosity: 0 # エラー以外はログに出さない
    # フルサービスリゾルバとして構築しないので、DNSSEC検証用のroot keyは導入しない
	#auto-trust-anchor-file: "/usr/local/etc/unbound/root.key"
	port: 53
	num-threads: 2 # CPUコア数と同数にする
	interface: 127.0.0.1 # 自身からのみ接続を受け付ける
	outgoing-range: 462 # 1024/cores - 50
	num-queries-per-thread: 231 # outgoing-range / 2
	so-reuseport: yes
	msg-cache-size: 512m # キャッシュサイズはお好みで
	msg-cache-slabs: 2 # slabs系は、num-threadsに近しい2の累乗にする
	rrset-cache-size: 1g # msg-cache-sizeの2倍にする
	rrset-cache-slabs: 2 # slabs系は、num-threadsに近しい2の累乗にする
	infra-cache-slabs: 2 # slabs系は、num-threadsに近しい2の累乗にする
	do-ip4: yes
	do-ip6: no # 契約しているISPがIPv6アドレスも払い出しているなら、yesにする
	do-udp: yes
	do-tcp: yes
	access-control: 0.0.0.0/0 refuse # Allow Listにしたいので、まずは全て拒否
	access-control: 127.0.0.0/8 allow # 自分自身からのアクセスを許可する
    # フルサービスリゾルバとしてUnboundを構築したいなら、ArchWiki等を参考にInterNICからnamed.cacheを取得し、Root hintsとして設定する
	#root-hints: "/usr/local/etc/unbound/root.hints"
	hide-identity: yes
	hide-version: yes
	cache-min-ttl: 3600
	prefetch: yes
	key-cache-size: 512m # キャッシュサイズはお好みで
	key-cache-slabs: 2 # slabs系は、num-threadsに近しい2の累乗にする
	neg-cache-size: 512m # キャッシュサイズはお好みで
	minimal-responses: yes
	rrset-roundrobin: yes
remote-control:
	control-enable: yes
	control-interface: 127.0.0.1
	server-key-file: "/usr/local/etc/unbound/unbound_server.key"
	server-cert-file: "/usr/local/etc/unbound/unbound_server.pem"
	control-key-file: "/usr/local/etc/unbound/unbound_control.key"
	control-cert-file: "/usr/local/etc/unbound/unbound_control.pem"
forward-zone:
    # LANにUnboundを建てているので、キャッシュヒットしなければ、そっちにフォワードさせる
	name:  "."
	forward-addr: 192.168.1.3
```

以上でUnboundを用いるための準備ができた。念のため、設定ファイルを適切に作成できているかを以下で確認する。

```
$ /usr/local/sbin/unbound-checkconf /usr/local/etc/unbound/unbound.conf
```

`no errors in ...` のように返されたらOK。エラーが返されたなら、その内容を読んで、typoしてないか等を確認する。

問題無さそうなら、以下でUnboundを再起動する。

```
# brew services restart unbound
```

Unboundを停止させ、PC起動時のサービス自動起動も止めたいなら、以下を実行する。

```
# brew services stop unbound
```

Unboundを止めたいが、PC起動時のサービス自動起動は残しておきたいなら、以下を実行する。

```
# brew services kill unbound
```

稼働しているかは、実際に問い合わせてみれば良い。

```
$ dig www.google.com @127.0.0.1
```

動作問題無さそうなら、 macOSのシステム環境設定 / ネットワークにて、DNSサーバに127.0.0.1を指定する。

## 参考
- [Unbound DNS server on macOS - sizeof(cat)](https://sizeof.cat/post/unbound-on-macos/)
- [Unbound - ArchWiki](https://wiki.archlinux.org/title/Unbound)
- [Performance Tuning — Unbound 1.14.0 documentation](https://unbound.docs.nlnetlabs.nl/en/latest/topics/performance.html)
- [DNSキャッシュサーバ チューニングの勘所](https://www.slideshare.net/hdais/dns-32071366)

