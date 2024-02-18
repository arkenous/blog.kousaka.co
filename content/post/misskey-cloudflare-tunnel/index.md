---
title: "MisskeyをCloudflare Tunnelで公開する"
date: 2024-02-18T11:40:00+09:00
description: "MisskeyをCloudflare Tunnelで公開することで、グローバルIPアドレスが変動する環境からでも、安全にサービスを提供可能にする。"
slug: cloudflare-tunnel-misskey
tags:
  - Cloudflare Tunnel
  - Misskey
keywords:
  - Cloudflare Tunnel
  - Misskey
  - Raspberry Pi
---

Cloudflare Tunnelを用い、MisskeyをCloudflareのネットワークを介して提供することで、サーバのIPアドレスを秘匿して直接的な攻撃を避けたり、Cloudflareが提供するWAF等の豊富なサービスを活用できる。また、これを使うことで一般的な家庭におけるグローバルIPアドレスが変動しうる環境下でも、サーバを外部に向けて容易に公開できる。

Cloudflare Tunnelを利用するにあたり、Cloudflareアカウントの作成と、用いるドメインのCloudflareへの登録が必要となるので、これを前提として以下を記載する。

# Misskeyセットアップ

公式で提供されているセットアップ手順に沿って作業する。手動で各ソフトウェアを入れる手順で進めている場合は、以下が不要となるため読み飛ばす。

* NGINX
* SSL証明書（Certbot）

無事Misskeyの起動が確認できたら、Cloudflare Tunnelの設定を行う。

# Cloudflare Tunnelセットアップ

Cloudflare Dashboardのサイドバーにある Zero Trust からCloudflare Oneに移動する。Cloudflare Oneのサイドバーにある Networks を展開すると Tunnels が表示されるので、これを選択する。

Tunnelsの管理ページが開かれたら、Tunnelの作成に進む。

Tunnelの接続に用いる方法として、Cloudflaredを選択する。

Tunnelの名称は、任意で命名する。

次に、Misskeyをホストするサーバに対し、Cloudflaredを導入する。Tunnel命名後、表示されるページ上に導入方法が示されていると思うので、これに沿って対応する。\
Docker ComposeでMisskeyを構築し、Cloudflare TunnelもDocker Composeで同様に稼働させる場合は、示された導入方法に含まれているトークン文字列を手元に控え、MisskeyのDocker Compose YAMLファイルにcloudflare/cloudflaredイメージでtunnel runを実行する記載を追加し、このenvironmentとして手元に控えたトークンを与える形で組み込む。

最後に、ルート情報の設定を求められるので、以下の通り設定する。

* Public hostname
  * Subdomain, Domain: Misskeyをホストする際のFQDN。サブドメインを用いない場合、Subdomainは空欄
  * Path: 空欄
* Service
  * Type: HTTP
  * URL: 127.0.0.1:3000

上記内容で保存後、Misskeyに設定したURLでWebブラウザよりアクセスすることで、自身が構築したMisskeyにアクセスできることを確認できるはず。
