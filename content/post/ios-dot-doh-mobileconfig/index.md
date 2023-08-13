---
title: "iOSでDNS over TLSやDNS over HTTPSを使う"
date: 2023-08-13T14:10:14+09:00
description: "iOSでDNS over TLSやDNS over HTTPSを使うための、mobileconfigの作成方法を残す。"
slug: ios-dot-doh-mobileconfig
---

iOS 16時点では、設定アプリからサーバアドレスを指定する形でのDNS over TLSやDNS over HTTPSの利用はできない。自分でDNSサーバを構築しており、iOS端末からこれらを利用したい場合は、mobileconfigファイルを作成して設定を流し込む必要がある。

[Configuring Multiple Devices Using Profiles](https://developer.apple.com/documentation/devicemanagement/configuring_multiple_devices_using_profiles)あたりのドキュメントを読みつつmobileconfigを作っていくのだろうが、[paulmillr/encrypted-dns](https://github.com/paulmillr/encrypted-dns)あたりを参考にしつつ書いてみて動いたので、備忘録として残す。

名前解決に特定のDNS over TLSサーバを用い、これを用いるかどうかを切り替えられるようにするには、以下のmobileconfigを作成する。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>DNSSettings</key>
            <dict>
                <key>DNSProtocol</key>
                <string>TLS</string>
                <key>ServerAddresses</key>
                <array>
                    <string>DNSサーバのIPアドレス</string>
                </array>
                <key>ServerName</key>
                <string>DNSサーバのFQDN</string>
            </dict>
            <key>PayloadDescription</key>
            <string>Configures device to use Encrypted DNS over TLS</string>
            <key>PayloadDisplayName</key>
            <string>My own DNS over TLS</string>
            <key>PayloadIdentifier</key>
            <string>com.apple.dnsSettings.managed.0766EE86-8DC9-4E4B-A441-7914594E5993</string>
            <key>PayloadType</key>
            <string>com.apple.dnsSettings.managed</string>
            <key>PayloadUUID</key>
            <string>92FED6B5-56DD-4684-982D-82FE8BE2A7F8</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>ProhibitDisablement</key>
            <false/>
        </dict>
    </array>
    <key>PayloadDescription</key>
    <string>Adds the DNS to Big Sur and iOS 14 based systems</string>
    <key>PayloadDisplayName</key>
    <string>My own DNS over TLS</string>
    <key>PayloadIdentifier</key>
    <string>DNSサーバのドメイン名.apple-dns</string>
    <key>PayloadRemovalDisallowed</key>
    <false/>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>DBDD1E7E-CEF1-435B-8E79-93A6824E9BBE</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

DNS over TLSではなくDNS over HTTPSで導入したい場合は、以下のように変更すると良い。

* DNSProtocolで指定する文字列を、TLSからHTTPSに変える
* ServerNameをServerURLに変更し、 `https://doh.example.com/dns-query` のような形式で接続先URLを指定する。

他の箇所については、

* Description周りは、自分にとってわかりやすい文言に変える
* IdentiferやPayloadUUIDで指定するUUIDは、手元で自前で都度生成したものを記載する

mobileconfigを作成できたら、AirDropなどでiOS端末に送る。受信すると取り込まれ、設定アプリからインストールできる。あとは、設定 / 一般 / VPNとデバイス管理 / DNS でDNS設定を切り替えられる。

---

この記事は、Tofu60 2.0 (70g Silicone bowl Gasket mount) + NuPhy Night Breeze + PBTfans Umbra で書きました。
