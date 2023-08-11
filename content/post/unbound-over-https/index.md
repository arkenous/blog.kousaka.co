---
title: "Unboundで構築したDNSフォワーダをDNS over HTTPSに対応させる"
date: 2022-12-04T17:28:32+09:00
description: "Unboundにover HTTPSを組み込む。"
slug: unbound-over-https
---

ひとつ前で書いた、DNS over TLSに対応させる設定の続き。

[Unboundで構築したDNSフォワーダをDNS over TLSに対応させ、プライベートDNSとして利用可能にする](https://blog.arkenous.net/unbound-over-tls/)

ほんの数行 `unbound.conf` に書き加えれば、DNS over HTTPSにも対応させられ、Google ChromeなどのSecure DNSとして利用できるので、設定しておく。

## Unboundへのover HTTPS機能の組み込み

over TLS機能は組み込み済みとする。組み込みをしておらず、SSL/TLS証明書の取得や、 `tls-service-key` 、 `tls-service-pem` の指定がまだであれば、上記記事を参考に対応を済ませておく。

over HTTPS機能を組み込むためには、 `unbound.conf` を以下のように編集する。

```conf
server:
    interface: 0.0.0.0@443
    interface: ::0@443
    
    # このあたりは、over TLS組み込み時に設定済みのはず
    tls-service-key: "/etc/letsencrypt/live/dns.example.com/privkey.pem"
    tls-service-pem: "/etc/letsencrypt/live/dns.example.com/fullchain.pem"
```

Unboundへの設定は、以上。

## 利用方法

WebブラウザのセキュアDNSなどで、以下のURLを指定する。

```
https://dns.example.com/dns-query
```

ドメイン名は、構築した自身のDNSサーバのものを指定する。

## 参考
- [DNS-over-HTTPS in Unbound](https://blog.nlnetlabs.nl/dns-over-https-in-unbound/)
- [DNS-over-HTTPS — Unbound 1.17.0 documentation](https://unbound.docs.nlnetlabs.nl/en/latest/topics/privacy/dns-over-https.html)

