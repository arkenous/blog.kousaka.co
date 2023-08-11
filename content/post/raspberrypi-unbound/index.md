---
title: "Raspberry PiとUnboundによるDNSフォワーダの構築"
date: 2022-06-04T15:07:28+09:00
description: "Raspberry PiにUnboundを入れてLANにDNSキャッシュサーバを構築し、案外時間を食う名前解決を高速化する方法を残す。"
slug: raspberrypi-unbound
---

DNSフォワーダをLAN内部に作成し、クライアントが問い合わせるDNSサーバをGoogle Public DNSなどのPublic DNSからこちらに変更することで、次の効果により名前解決を高速化し、Webブラウジング等をより快適にすることを目指す。

- 自分専用のDNSフォワーダを用意し問い合わせ結果をキャッシュさせることで、キャッシュヒット率を上げる
- なるべくLAN内部で名前解決が完結するようにし、インターネット越しに名前解決することによる、距離的な時間のロスを減らす

[Unbound](https://nlnetlabs.nl/projects/unbound/about/)を用いてDNSフォワーダを構築する。上記目的を踏まえ、クライアントからの問い合わせを受けてルートドメインから反復問い合わせするフルサービスリゾルバを構築するのではなく、あくまでキャッシュヒットしなかった問い合わせを別のフルサービスリゾルバ （Google Public DNSなど）に中継するフォワーダとして構築する。

## Raspberry Piのセットアップ

Raspberry Pi本体をどこかしらから調達する。最近のRaspberry Piであれば64bit CPUが搭載されているので、導入するRaspberry Pi OSを選ぶ際、可能であれば64bit OSを選ぶ。また、サーバ用途で利用する場合、GUIは不要なので、Liteバージョンを選ぶとハードウェア資源を有効活用できる。

Raspberry Pi自体のセットアップは割愛する。TimeZoneの設定などを忘れずに。

ネットワークにWi-Fi経由で接続できるモデルもあるが、接続の安定性・通信速度の観点からなるべくEthernet（有線接続）経由でネットワークに接続させることが望ましい。加えて、サーバ用途であるため、IPアドレスも固定させておく必要がある。大抵の場合、LANを構築するルータがDHCPサーバを兼ねており、クライアントに対してプライベートIPアドレスが自動的に払い出される状態になっていると思う。その場合は、 `/etc/dhcpcd.conf` を編集し、DHCPクライアントの設定により固定のIPアドレスを用いるよう対応する。設定ファイル中に設定例が豊富にコメントされているので、それらを見てもらえれば理解できると思うが、[Debian Wiki](https://wiki.debian.org/DHCP_Client)や[ArchWiki](https://wiki.archlinux.org/title/Dhcpcd)も参考になる。

```
interface eth0
static ip_address=192.168.1.3/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

`domain_name_servers` については、UnboundによるDNSキャッシュサーバを構築後、そちらを指すように変更するが、ひとまず現状はGoogle Public DNSを指定しておく。

ルータの設定でも、Raspberry Piに対し上記指定したIPアドレスを固定で払い出すよう、設定しておく。

## Unboundの導入

Raspberry Piがインターネットに疎通したら、Unboundを導入する。Raspberry Pi OSはDebianベースなので、apt-getでインストール。

```
# apt-get install unbound
```

導入できたら、 `/etc/unbound/unbound.conf` に対して設定を加えていく。設定のドキュメントは `/usr/share/doc/unbound/examples/unbound.conf` にあるので、そちらを参照すること。僕自身は、以下のように設定した。

```conf
# Unbound configuration file for Debian.
#
# See the unbound.conf(5) man page.
#
# See /usr/share/doc/unbound/examples/unbound.conf for a commented
# reference config file.
#
# The following line includes additional configuration files from the
# /etc/unbound/unbound.conf.d directory.
include-toplevel: "/etc/unbound/unbound.conf.d/*.conf"

server:
	verbosity: 0 # エラー以外はログに出さない
	num-threads: 4 # CPUコア数と同数にする
	interface: 0.0.0.0 # 接続を受け付けるIPアドレスは制限しない
	outgoing-range: 206 # 1024/cores - 50
	num-queries-per-thread: 103 # outgoing-range / 2
	so-reuseport: yes
	msg-cache-size: 512m # キャッシュサイズはお好みで
	msg-cache-slabs: 4 # slabs系は、num-threadsに近しい2の累乗にする
	rrset-cache-size: 1g # msg-cache-sizeの2倍にする
	rrset-cache-slabs: 4 # slabs系は、num-threadsに近しい2の累乗にする
	infra-cache-slabs: 4 # slabs系は、num-threadsに近しい2の累乗にする
	do-ip4: yes
	do-ip6: no # 契約しているISPがIPv6アドレスも払い出しているなら、yesにする
	do-udp: yes
	do-tcp: yes
	access-control: 0.0.0.0/0 refuse # Allow Listにしたいので、まずは全て拒否
	access-control: 127.0.0.0/8 allow # 自分自身からのアクセスを許可する
	access-control: 192.168.1.0/24 allow # LANに属するクライアントからのアクセスを許可する
    # フルサービスリゾルバとしてUnboundを構築したいなら、ArchWiki等を参考にInterNICからnamed.cacheを取得し、Root hintsとして設定する
	# root-hints: root.hints 
	hide-identity: yes
	hide-version: yes
	cache-min-ttl: 3600
	prefetch: yes
	key-cache-size: 512m # キャッシュサイズはお好みで
	key-cache-slabs: 4 # slabs系は、num-threadsに近しい2の累乗にする
	neg-cache-size: 512m # キャッシュサイズはお好みで
	minimal-responses: yes
	rrset-roundrobin: yes
    # DNSで特定ドメインをNXDOMAINにしブロックしたいなら、Unbound用（local-zone:で定義されているもの）を拾ってきて、以下のように定義する
    # https://github.com/openwrt/packages/tree/master/net/adblock/files あたりから探すと良い
	# include: /etc/unbound/adblock-rules.conf
forward-zone:
	# キャッシュにヒットしなかったら、Google Public DNSに問い合わせをフォワードする
	name: "."
	forward-addr: 8.8.8.8
	forward-addr: 8.8.4.4
```

パフォーマンスチューニングに際しては、次の記事が参考になる。

[Performance Tuning — Unbound 1.14.0 documentation](https://unbound.docs.nlnetlabs.nl/en/latest/topics/performance.html)

[DNSキャッシュサーバ チューニングの勘所](https://www.slideshare.net/hdais/dns-32071366)

以上でUnboundの設定が完了となる。以下コマンドにより、設定ファイルの記載内容に誤りがないかを確認する。

```
$ unbound-checkconf /etc/unbound/unbound.conf
```

`no errors in...` のように返ってきたら問題ない。エラーが返されたら、内容を確認して、typoがないか等を調べる。

設定に問題が無いようなら、以下でUnboundを立ち上げる。

```
# service unbound start
```

正常に起動したかを以下で調べる。

```
$ service unbound status
```

以下コマンドにより実際に名前解決の問い合わせをUnboundに投げて、名前解決されるかを確認する。

```
$ dig www.google.com @192.168.1.3
```

上記は、同一LANに属するクライアントから問い合わせを投げる例。Raspberry Pi内部で動作確認する際は、 `@127.0.0.1` とする。

## Unboundを用いるよう設定

正常に名前解決できることが確認できたら、このUnboundをDNSサーバとして用いるよう、各クライアントで設定する。

クライアントで設定する前に、Raspberry Pi自身が用いるDNSサーバとして、このUnboundを用いるよう設定する。本記事冒頭でdhcpcdの設定をした際、 `domain_name_servers` にGoogle Public DNSを指定したが、これをUnboundに変更する。

```
# /etc/dhcpcd.conf

static domain_name_servers=127.0.0.1
```

次に、各クライアントのネットワーク設定で、DNSサーバとしてUnboundを用いるよう設定変更する。個別にRaspberry PiのIPアドレスを指定しても良いが、ルータのDHCPサーバ設定からDNSサーバアドレスを指定する形にしても良い。

---

以上で、対応完了となる。初めてアクセスするドメインの場合は、問い合わせをフォワードする都合上、今までと名前解決に要する時間は変わらない。だが、問い合わせ結果がキャッシュされてからは、ドメインのTTL（キャッシュの有効期限）以内であれば、LAN内部でより迅速に名前解決できる。

## 参考
- [Install Raspberry Pi OS Bullseye on Raspberry Pi (Illustrated guide) | Raspberry tips](https://raspberrytips.com/install-raspbian-raspberry-pi/)
- [DHCP_Client - Debian Wiki](https://wiki.debian.org/DHCP_Client)
- [dhcpcd - ArchWiki](https://wiki.archlinux.org/title/Dhcpcd)
- [Unbound - ArchWiki](https://wiki.archlinux.org/title/Unbound)
- [Performance Tuning — Unbound 1.14.0 documentation](https://unbound.docs.nlnetlabs.nl/en/latest/topics/performance.html)
- [DNSキャッシュサーバ チューニングの勘所](https://www.slideshare.net/hdais/dns-32071366)
- [packages/net/adblock/files at master · openwrt/packages](https://github.com/openwrt/packages/tree/master/net/adblock/files)

