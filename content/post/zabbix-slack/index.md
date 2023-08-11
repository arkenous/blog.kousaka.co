---
title: "Zabbix + Slackによる障害通知システムの構築ログ"
date: 2015-11-07T08:00:00+09:00
description: "自宅サーバの稼動状態を監視するために，Zabbix + Slackによる監視環境を構築する．"
slug: zabbix-slack
---

自宅サーバの稼動状態を監視するために，Zabbix + Slackによる監視環境を構築する．Zabbixを用いた監視サーバにはConoHaのVPSを利用する．サーバの動作OSはいずれもArchLinuxである．

## ConoHa : Zabbixサーバ

Zabbixを使うために，まずはLAMP環境を構築する必要がある．

### LAMP環境の構築

インストールするパッケージは以下のもの．

- Apache
- MariaDB
- PHP
- PHP-Apache

以下のコマンドを実行し，各種パッケージをインストールする．

```
$ yaourt -S apache php php-apache mariadb
```

インストールが完了すれば，まずはApacheの設定を行う．

#### Apache

Apacheの設定ファイルであるhttpd.confは/etc/httpd/conf/にあるので，これを編集する．今回の運用ではhttpsのみ必要であるため，必要のないServerNameおよびListen 80の部分をコメントアウトしておいた．OpenSSLを用いたSSL対応を行うために，以下のコマンドを/etc/httpd/confの下で順に実行し，SSL証明書を発行する．

```
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key
# chmod 600 server.key
# openssl req -new -key server.key -out server.csr
# openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

発行が完了したら，httpd.confを編集し，ApacheをSSLに対応させる．必要なのは，以下の3行である．コメントアウトされている場合はアンコメントする．

```
LoadModule ssl_module modules/mod_ssl.so
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
Include conf/extra/httpd-ssl.conf
```

編集できたら，以下のコマンドを実行してApacheの起動とサーバの起動時にApacheを自動起動するようにしておく．

```
# systemctl start httpd
# systemctl enable httpd
```

#### PHP

次は，Apache上でPHPを動かせるようにhttpd.confを編集する．
まずは，以下の1行

```
LoadModule mpm_event_module modules/mod_mpm_event.so
```

を，

```
LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
```

のように書き換える．また，以下の1行をLoadModule行の最後に追加する．

```
LoadModule php5_module modules/libphp5.so
```

さらに，以下の1行をInclude行の最後に追加する．

```
Include conf/extra/php5_module.conf
```

#### MariaDB

ArchLinuxにおけるMySQLのデフォルト実装であるMariaDBの設定を行う．まずは以下のコマンドを順に実行し，mysqldを起動する．

```
# mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
# systemctl start mysqld
```

起動できたら，以下のコマンドでデータベースの初期設定を行う．

```
# mysql_secure_installation
```

最初に聞かれるroot passwordについては，何も入力せずにEnterを押せばいい．また，このステップにて設定するrootアカウントのパスワードは後で使うので絶対に忘れないこと．

設定が完了したら，ネットワーク越しにMySQLにログインされないように，/etc/mysql/の中にあるmy.cnfの以下の行をアンコメントする．

```
skip-networking
```

アンコメントできたら，以下の二つのコマンドを実行し，mysqldの再起動とサーバ起動時に自動的にmysqldが起動するようにする．

```
# systemctl restart mysqld
# systemctl enable mysqld
```

#### Zabbix

以下のコマンドを実行し，Zabbixをインストールする．

```
$ yaourt -S zabbix-server-mysql
```

インストールが完了したら，Zabbixの設定を行う．まずは，ApacheからZabbixにアクセスさせるための設定ファイルを，/etc/httpd/conf/extra/の下にzabbix.confとして作成する．

```
NameVirtualHost *:443
<IfModule mod_alias.c>
    Alias /zabbix /usr/share/webapps/zabbix/
</IfModule>

<Directory /usr/share/webapps/zabbix>
    Options -Indexes +FollowSymlinks +ExecCGI
    AllowOverride all
    Require all granted
</Directory>

<VirtualHost *:443>
    ServerAdmin foo@foovar.com
    DocumentRoot "/usr/share/webapps/zabbix"
    ServerName zabbix.your.domain
    ErrorLog /var/log/httpd/zabbix.your.domain-error_log
    CustomLog /var/log/httpd/zabbix.your.domain-access_log common
    
    SSLEngine on
    SSLCertificateFile "/etc/httpd/conf/server.crt"
    SSLCertificateKeyFile "/etc/httpd/conf/server.key"
</VirtualHost>
```

ファイルを作成できたら，httpd.confのInclude行の最後に以下の行を追加し，この設定ファイルを読み込ませる．

```
Include conf/extra/zabbix.conf
```

次に，/etc/php/内のphp.iniを編集して，Zabbixの動作に必要なPHPの設定を行う．以下に挙げるような設定を行った．

```
max_execution_time = 300
max_input_time = 300
post_max_size = 16M
always_populate_raw_post_data = -1
extension = bcmath.so
extension = gd.so
extension = mysqli.so
extension = pdo_mysql.so
extension = sockets.so
date.timezone = "Asia/Tokyo"
```

PHPの設定が完了したら，次はデータベースの設定を行う．MySQLに，以下のコマンドでログインする．

```
$ mysql -u root -p
```

パスワードを聞かれると思うので，MySQLインストール後の初期設定にて設定したrootアカウントのパスワードを入力する．
ログインできたら以下のコマンドを順に実行し，Zabbixに関係するデータベースを扱うためのユーザを作成する．

```
create database zabbix;
create user zabbix identified by 'input password here';
grant all on zabbix.* to zabbix@localhost identified by "input password here";
```

コマンドが実行できたら`exit;`などでMySQLからログアウトし，次のコマンドを順に実行してZabbixのテンプレートをインポートする．

```
$ mysql -u zabbix -p zabbix < /usr/share/zabbix/database/schema.sql
$ mysql -u zabbix -p zabbix < /usr/share/zabbix/database/images.sql
$ mysql -u zabbix -p zabbix < /usr/share/zabbix/database/data.sql
```

実行のたびにパスワードを聞かれると思うが，ここではデータベース作成時に権限を割り当てたzabbixユーザのパスワードを入力する．次に，Zabbixサーバの設定を行う．/etc/zabbix/の中にあるzabbix_server.confの編集を行う．以下のように編集する．

```
DBHost = localhost
DBName = zabbix
DBUser = zabbix
DBPassword=zabbix's password
```

編集が完了したら，zabbix-serverの起動と，サーバ起動時にzabbix-serverを自動起動する設定を行う．

```
# systemctl start zabbix-server
# systemctl enable zabbix-server
```

また，後に説明する監視対象サーバ（エージェント）からの通信を受け取るための10051番ポートが開放されているかを確認する．
正常に起動できたら，設定したURLでZabbixにアクセスし，表示されるインストールウィザードを進めていく．ちなみに，インストールウィザードの最初に聞かれるログインユーザ名とパスワードは，Zabbixのデフォルト設定であるユーザ名のAdmin，パスワードはzabbixでログインする．

## 自宅サーバ : エージェント（クライアント）

監視対象となるエージェントサーバには，zabbix-agentのみをインストールする．

```
$ yaourt -S zabbix-agent
```

インストールできたら，/etc/zabbix/にあるzabbix_agentd.confを編集して設定を行う．以下の部分を設定した．

```
Server = ZabbixサーバのIPアドレス
ServerActive = ZabbixサーバのIPアドレス
Hostname = このサーバのホスト名
```

設定が完了したら，ファイアウォールにてZabbixサーバからの通信を受け取るのに必要な10050番ポートが適切に開かれていることを確認し，以下のコマンドを実行して起動とサーバ起動時に自動起動する設定を行う．

```
# systemctl start zabbix-agentd
# systemctl enable zabbix-agentd
```

## Slackとの連携

Zabbixで監視しているサーバになんらかの障害が発生した際に，アラートをSlackに飛ばす設定をする．まず，アラートを飛ばしたいSlackのチームにおいてIncoming WebHooksを設定し，Webhook URLを控えておく．次に以下に示すようなスクリプトを作成し，実行権限を付与して/usr/share/zabbix/alertscripts/に置いておく．

```sh
#!/usr/bin/bash

# Current time
time="$1"

# Host name
host="$2"

# Current trigger value. PROBLEM or OK
status="$3"

# Numerical trigger severity. 0: Not classified, 1: Information, 2: Warning, 3: Average, 4: High, 5: Disaster
nseverity="$4"

# Trigger severity name
severity="$5"

# Name of the trigger
name="$6"

# Trigger URL
url="$7"

# Slack channel name to post
channel="$8"

if [ $status = 'OK' ]; then
    color='#59DB8F'
elif [ $nseverity = '0' ]; then
    color='#97AAB3'
elif [ $nseverity = '1' ]; then
    color='#7499FF'
elif [ $nseverity = '2' ]; then
    color='#FFC859'
elif [ $nseverity = '3' ]; then
    color='#FFA059'
elif [ $nseverity = '4' ]; then
    color='#E97659'
elif [ $nseverity = '5' ]; then
    color='#E45959'
else
    color='#000000'
fi


#{
#    "channel":"${channel}",
#    "attachments":[
#        {
#            "fallback":"${status} on ${host} at ${time}",
#            "pretext":"${status} on ${host} at ${time}",
#            "color":"${color}",
#            "fields":[
#                {
#                    "title":"[${severity}] ${name}",
#                    "value":"${url}",
#                    "short":false
#                }
#            ]
#        }
#    ]
#}

payload="payload={\"channel\":\"${channel}\",\"attachments\":[{\"fallback\":\"${status} on ${host} at ${time}\",\"pretext\":\"${status} on ${host} at ${time}\",\"color\":\"${color}\",\"fields\":[{\"title\":\"[${severity}] ${name}\",\"value\":\"${url}\",\"short\":false}]}]}"

curl -m 5 --data-urlencode "${payload}" https://hooks.slack.com/services/YOUR/SLACK/INCOMING/WEBHOOK/URL
```

配置できたら，ZabbixでActionの作成を行う．ConfigurationのActionsにてEvent SourceをTriggersにしたアクションを作成し，OperationsタブでOperation typeにRemote commandを，Target listにはCurrent hostを，TypeにCustom scriptを，Execute onにZabbix serverを指定する．次にCommandsの部分には以下のようなものを設定する．

```
/usr/share/zabbix/alertscripts/zabbixslack.sh "{TRIGGER.STATUS}" "{TRIGGER.NAME}" "{ITEM.NAME1} {ITEM.KEY1}"
```

これらの設定を行い，Actionを追加する．これで，障害発生をZabbixが検知したら，Slackに通知が飛ぶはず．

