---
title: "MastodonをCloudflare Tunnelで公開する"
date: 2024-02-18T14:00:00+09:00
description: "MastodonをCloudflare Tunnelで公開することで、グローバルIPアドレスが変動する環境からでも、安全にサービスを提供可能にする。"
slug: cloudflare-tunnel-mastodon
tags:
  - Cloudflare Tunnel
  - Mastodon
keywords:
  - Cloudflare Tunnel
  - Mastodon
  - Raspberry Pi
---

Cloudflare Tunnelを用い、MastodonをCloudflareのネットワークを介して提供することで、サーバのIPアドレスを秘匿して直接的な攻撃を避けたり、Cloudflareが提供するWAF等の豊富なサービスを活用できる。また、これを使うことで一般的な家庭におけるグローバルIPアドレスが変動しうる環境下でも、サーバを外部に向けて容易に公開できる。

Cloudflare Tunnelを利用するにあたり、Cloudflareアカウントの作成と、用いるドメインのCloudflareへの登録が必要となるので、これを前提として以下を記載する。

なお、本記事でFQDNと示している部分について、ドメイン末尾に付与する、ルートを示す `.` は含めないこととする。

# Mastodonセットアップ
公式リポジトリからdocker-compose.ymlを取得し、構築・設定作業を行う。詳細な手順についてはここでは触れない。NGINXやSSL証明書は不要。\
非Docker環境をDocker環境に移行する際は、以下記事などが参考になる。Cloudflare Tunnelを用いない場合と同様にドメイン名等を指定して設定していく。

[Mastodonの非Docker→Docker化](https://zenn.dev/yakumo/articles/e0304a2ed4bc453e879b2dabf906cad7)

無事Mastodonをセットアップできたら、Cloudflare Tunnelの設定を行う。

# Cloudflare Tunnelセットアップ

Cloudflare Dashboardのサイドバーにある Zero Trust からCloudflare Oneに移動する。Cloudflare Oneのサイドバーにある Networks を展開すると Tunnels が表示されるので、これを選択する。

Tunnelsの管理ページが開かれたら、Tunnelの作成に進む。

Tunnelの接続に用いる方法として、Cloudflaredを選択する。

Tunnelの名称は、任意で命名する。

次に、Mastodonをホストするサーバに対し、Cloudflaredを導入する。Tunnel命名後、表示されるページ上に導入方法が示されていると思うので、これに沿って対応する。\
Docker ComposeでMastodonを構築し、Cloudflare TunnelもDocker Composeで同様に稼働させる場合は、示された導入方法に含まれているトークン文字列を手元に控え、MastodonのDocker Compose YAMLファイルにcloudflare/cloudflaredイメージでtunnel runを実行する記載を追加し、このEnvironmentとして手元に控えたトークンを与える形で組み込む。

最後に、ルート情報の設定を求められるので、一旦以下で設定し保存する。

* Public hostname
  * Subdomain, Domain: Mastodonをホストする際のFQDN。サブドメインを用いない場合、Subdomainは空欄
  * Path: api/v1/streaming
* Service
  * Type: HTTP
  * URL: 127.0.0.1:4000

保存後、作成済みのTunnelがリストアップされるので、先ほど作成したTunnelの右端の三点ボタンから Configure を選び設定画面に移る。

設定画面上部の Public Hostname を選び、api/v1/streamingをPathとして指定したものの右端の三点ボタンから Edit を選び編集画面に移る。

改めて、以下の通り設定する。なお、HTTP Host Headerは Additional application settings を展開し、さらに HTTP Settings を展開することで表示される。

* Public hostname
  * Subdomain, Domain: Mastodonをホストする際のFQDN。サブドメインを用いない場合、Subdomainは空欄
  * Path: api/v1/streaming
* Service
  * Type: HTTP
  * URL: 127.0.0.1:4000
* HTTP Host Header: FQDN

設定し保存したら、Public hostnamesのリストに戻るはず。ここで、 Add a public hostname を選び追加でPublic Hostnameを設定する。これには、以下の設定を行う。

* Public hostname
  * Subdomain, Domain: Mastodonをホストする際のFQDN。サブドメインを用いない場合、Subdomainは空欄
  * Path: 空欄
* Service
  * Type: HTTP
  * URL: 127.0.0.1:3000
* HTTP Host Header: FQDN

上記で設定し保存後、Mastodonに設定したURLでWebブラウザよりアクセスすることで、自身が構築したMastodonにアクセスできることを確認できるはず。
