---
title: "ownCloudの構築ログ"
date: 2016-01-09T17:00:00+09:00
description: "ArchLinuxにおいてownCloudを構築したので，そのログを書き殘そうと思う．"
slug: owncloud
---

ArchLinuxにおいてownCloudを構築したので，そのログを書き殘そうと思う．

## LAMP

まずはArchLinuxにLAMP環境を構築する．
以下のコマンドを実行してまとめてインストールする．

```
$ yaourt -S apache php php-apache mariadb
```

インストールが完了したら，Apacheの設定から行っていく．

### Apache

Apacheの基本的な設定ファイルである`httpd.conf`は`/etc/httpd/conf`の下にある．これを編集する．今回はhttpsのみでアクセスすることを想定しているので，必要のない`ServerName`および`Listen 80`の部分をコメントアウトして保存する．OpenSSLを用いたSSL対応を行うために，以下のコマンドを`/etc/httpd/conf`の下で順に実行し，SSL証明書を発行する．

```
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key
# chmod 600 server.key
# openssl req -new -key server.key -out server.csr
# openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

発行が完了したら`httpd.conf`を編集し，ApacheをSSLに対応させる．
必要なのは以下の3行である．コメントアウトされている場合はアンコメントする．

```
LoadModule ssl_module modules/mod_ssl.so
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
Include conf/extra/httpd-ssl.conf
```

また，`/etc/httpd/conf/extra`の下にある`httpd-ssl.conf`を編集する．`Listen 443`となっているかを確認し，`<VirtualHost _default_:443>`內の`ServerName`を適切に設定しておく．

編集できたら，以下のコマンドを実行してApacheを起動し，サーバの起動時にApacheが自動起動するようにしておく．

```
# systemctl start httpd
# systemctl enable httpd
```

### PHP

次は，Apache上でPHPを動かせるように`httpd.conf`を編集する．
まずは，以下の1行

```
LoadModule mpm_event_module modules/mod_mpm_event.so
```

を，

```
LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
```

のように書き換えて有効にする．また，以下の1行を`LoadModule`行の最後に追記する．

```
LoadModule php5_module modules/libphp5.so
```

さらに，以下の1行を`Include`行の最後に追記する．

```
Include conf/extra/php5_module.conf
```

### MariaDB

ArchLinuxにおけるMySQLのデフォルト実裝であるMariaDBの設定を行う．
まずは以下のコマンドを実行し，`mysqld`を起動する．

```
# mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
# systemctl start mysqld
```

起動できたら，以下のコマンドでデータベースの初期設定を行う．

```
# mysql_secure_installation
```

最初に聞かれるroot passwordについては，何も入力せずにEnterを押せばいい．また，このステップにて設定するMySQL用のrootアカウントのパスワードは後で使うので絶対に忘れないこと．設定が完了したら，ネットワーク越しにMySQLにログインされないように，`/etc/mysql`の下にある`my.cnf`の以下の行をアンコメントする．

```
skip-networking
```

アンコメントできたら以下のコマンドを実行し，`mysqld`の再起動とサーバ起動時に自動起動する設定をする．

```
# systemctl restart mysqld
# systemctl enable mysqld
```

以上でLAMP環境は構築できた．いよいよownCloudの構築となる．

## ownCloud

まずは，ownCloudのパッケージを以下のコマンドでインストールする．

```
$ yaourt -S owncloud
```

また，ownCloudで利用するPHPの拡張パッケージもインストールする．

```
$ yaourt -S php-intl php-mcrypt
```

インストールができたら，`/etc/php/php.ini`內の以下のextensionをアンコメントして有効にする．

```
gd.so
iconv.so
xmlrpc.so
zip.so
bz2.so
curl.so
intl.so
mcrypt.so
openssl.so
pdo_mysql.so
mysql.so
```

次に，ownCloudのサンプル設定ファイルをApacheの設定ディレクトリにコピーする．

```
# cp /etc/webapps/owncloud/apache.example.conf /etc/httpd/conf/extra/owncloud.conf
```

コピーしたら，`httpd.conf`內の`Include`行の最後に以下の行を追記する．

```
Include conf/extra/owncloud.conf
```

また，`httpd.conf`內の以下の行をコメントアウトして無効化する．

```
LoadModule dav_module modules/mod_dav.so
LoadModule dav_fs_module modules/mod_dav_fs.so
```

コメントアウトできたら，次はownCloud本體のディレクトリにApacheからアクセスできるように権限の設定を行う．以下のコマンドを実行する．

```
# chown -R http:http /usr/share/webapps/owncloud
```

権限設定の完了後，Apacheの再起動を行う．

```
# systemctl restart httpd
```

ownCloudのファイル管理にはデータベースが必要なので，ownCloud用のデータベースとユーザを用意する．まずは以下のコマンドを実行してMySQLにログインする．

```
$ mysql -u root -p
```

ログインできたら，以下の三つのSQL文を順に実行し，ownCloud用のデータベースとユーザの作成を行う．

```
create database owncloud;
create user owncloud identified by 'input password here';
grant all privileges on owncloud.* to 'owncloud'@'localhost' identified by 'input password here';
```

データベース，ユーザの作成ができたら以下のコマンドでMySQLの再起動を行う．

```
# systemctl restart mysqld
```

これでownCloudの構築ができた．`https://ServerNameで指定したドメイン名/owncloud`というURLでアクセスすれば，ownCloudの初期設定畫面が表示されるはず．データベースユーザやデータベースパスワードはMySQLの構築時に設定したものを指定する．データベースホストは`localhost`となる．データディレクトリは`/usr/share/webapps/owncloud/data`となる．ownCloudのユーザ名とパスワードはもちろんお好みで．これら設定が完了すれば，いよいよownCloudが利用可能となるはずである．

