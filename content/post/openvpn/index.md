---
title: "OpenVPN環境の構築"
date: 2016-02-20T18:00:00+09:00
description: "iPhoneのモバイル回線からでもVPNとプロキシ経由でWebブラウジングできるように，Arch Linux上にOpenVPN Serverを構築した．"
slug: openvpn
---

iPhoneのモバイル回線からでもVPNとプロキシ経由でWebブラウジングできるように，Arch Linux上にOpenVPN Serverを構築した．

## OpenVPN Serverの構築

まずは以下のコマンドを実行し，OpenVPN Server構築に必要なパッケージをインストールする．

```
$ yaourt -S openvpn easy-rsa
```

次に，先ほどインストールした`easy-rsa`を用いてOpenVPNに必要なPKI（公開鍵基盤）を構築する．以下のコマンドを実行して，`easy-rsa`を`/root`の下に置く．

```
# cp -r /usr/share/easy-rsa /root/
```

`easy-rsa`を`/root`の下に置けたら，PKIを構築する．

### PKI（公開鍵基盤）の構築

以下のコマンドを順に実行していき，認証局の構築およびサーバ証明書の発行，Diffie-Hellman鍵交換の際に用いる暗号キーファイルの作成，TLSキーファイルの作成を行う．

```
# cd /root/easy-rsa
# source ./vars
# ./clean-all
# ./build-ca
# ./build-key-server openvpnsv
# ./build-dh
# openvpn --genkey --secret /etc/openvpn/ta.key
```

各種ファイルが作成できたら，サーバ設定に必要なファイルを`/etc/openvpn`の下に置く．

```
# cp keys/ca.crt /etc/openvpn/
# cp keys/openvpnsv.crt /etc/openvpn/
# cp keys/openvpnsv.key /etc/openvpn/
# cp keys/dh2048.pem /etc/openvpn/
```

ファイル作成ができたら，OpenVPN Serverの設定ファイルを編集していく．

### OpenVPN Server設定ファイルの作成

まずは，以下のコマンドを実行して設定ファイルのサンプルをコピーする．

```
# cp /usr/share/openvpn/examples/server.conf /etc/openvpn/server.conf
```

コピーできたら，設定ファイルを編集する．以下に`server.conf`のdiffを載せる．

```diff
$ diff /usr/share/openvpn/examples/server.conf /etc/openvpn/server.conf
35,36c35,36
< ;proto tcp
< proto udp

> proto tcp
> ;proto udp
79,80c79,82
< cert server.crt
< key server.key  # This file should be kept secret

> #cert server.crt
> cert openvpnsv.crt
> #key server.key  # This file should be kept secret
> key openvpnsv.key  # This file should be kept secret
192a195
> push "redirect-gateway def1"
201a205
> push "dhcp-option DNS 8.8.8.8"
209c213
< ;client-to-client

> client-to-client
244c248
< ;tls-auth ta.key 0 # This file is secret

> tls-auth ta.key 0 # This file is secret
267,268c271,272
< ;user nobody
< ;group nobody

> user nobody
> group nobody
304a309,311
>
> push "dhcp-option PROXY_HTTP your.proxy.server port"
> push "dhcp-option PROXY_HTTPS your.proxy.server port"
```

また，編集後のコメント部分を除いた`server.conf`を以下に載せる．

```
port 1194
proto tcp
dev tun

ca ca.crt
cert openvpnsv.crt
key openvpnsv.key
dh dh2048.pem

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt

push "redirect-gateway def1"
push "dhcp-option DNS 8.8.8.8"
client-to-client

keepalive 10 120

tls-auth ta.key 0 # This file is secret

comp-lzo

user nobody
group nobody

persist-key
persist-tun

push "dhcp-option PROXY_HTTP your.proxy.server port"
push "dhcp-option PROXY_HTTPS your.proxy.server port"
```

この設定により，クライアントからの全てのトラフィックがOpenVPNを通して流れ，HTTPやHTTPSのトラフィックは`dhcp-option PROXY_HTTP`や`dhcp-option PROXY_HTTPS`で指定したプロキシを通してアクセスされる．

次に，OpenVPN用のネットワークインタフェースから外部と接続できる通常のネットワークインタフェースへトラフィックをフォワーディングする設定を行う．`/etc/sysctl.d/30-ipforward.conf`を作成し，以下の行を追加する．

```
# Enable packet forwarding
net.ipv4.ip_forward=1
```

> ネットワーク設定に`systemd-networkd`を用いている場合，バージョンによってはこれが上記設定を上書きしてフォワーディングをオフにしてしまうため，`foo.network`のような`systemd-networkd`インタフェース設定ファイルの`[Network]`セクションで`IPForward=kernel`を追加する必要がある．
> [https://wiki.archlinuxjp.org/index.php/%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E5%85%B1%E6%9C%89](https://wiki.archlinuxjp.org/index.php/%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E5%85%B1%E6%9C%89)

作成できたらサーバの再起動を行い，フォワーディングの設定を反映させる（再起動しないと設定が反映されない）．また，ufwの設定を変更し，フォワーディングの許可およびフォワードルールの設定を行う．まずは`/etc/default/ufw`を以下のように設定する．

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

また，`/etc/ufw/before.rules`のヘッダ部分と`*filter`部分の間に以下の行を追加し，フォワードルールの設定を行う．

```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to enp0s17
-A POSTROUTING -s 10.8.0.0/24 -o enp0s17 -j MASQUERADE

# don't delete the "COMMIT" line or the NAT table rules above won't be processed
COMMIT
# END OPENVPN RULES
```

設定ができたら以下のコマンドを実行し，ufwサービスの再起動およびにOpenVPN用のポート開放を行う．

```
# systemctl restart ufw
# ufw allow 1194
```

そして，以下のコマンドでOpenVPN Serverの起動と，サーバ起動時にサービスが自動起動するように設定する．

```
# systemctl start openvpn@server.service
# systemctl enable openvpn@server.service
```

サービスのステータスは，以下のコマンドで確認できる．

```
$ systemctl status system-openvpn.slice
```

## OpenVPN Clientの設定

クライアント用の設定ファイルを作成する．まず，サーバで以下のコマンドを実行し，クライアント側の証明書ファイルの発行を行う．以下の例ではパスワード認証無しの証明書を作成しているが，`build-key`の代わりに`build-key-pass`を用いることでパスワード認証のある証明書を作成できる．

```
# cd /root/easy-rsa
# source ./vars
# ./build-key client-ios
```

PKI構築で作成したファイルの中で，CAの証明書`ca.crt`とTLSキーファイル`ta.key`，クライアントの証明書`client-ios.crt`と鍵ファイル`client-ios.key`を用いる．設定ファイル例`ios.ovpn`を以下に載せる．

```
client
dev tun
proto tcp
remote your.openvpn.server 1194
resolv-retry infinite
nobind
persist-key
persist-tun
user nobody
group nobody
remote-cert-tls server
comp-lzo
redirect-gateway def1

<ca>
-----BEGIN CERTIFICATE-----
WRITE YOUR ca.crt CERTIFICATE HERE
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
WRITE YOUR client-ios.crt CERTIFICATE HERE
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
WRITE YOUR client-ios.key PRIVATE KEY HERE
-----END PRIVATE KEY-----
</key>
key-direction 1
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
WRITE YOUR ta.key OpenVPN STATIC KEY HERE
-----END OpenVPN Static key V1-----
</tls-auth>
```

上のような設定ファイルを作成したら，iPhoneにOpenVPN Connectアプリをインストールし，PCで起動したiTunesを経由してこのアプリに設定ファイルを転送し，OpenVPN Connectアプリを起動して設定ファイルをインポートすれば，OpenVPNサーバに接続できるはずだ．

