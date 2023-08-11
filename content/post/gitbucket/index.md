---
title: "Arch LinuxにGitBucketをインストールしたログ"
date: 2015-02-22T11:00:00+09:00
description: "プライベートなGitホスティングサーバを持っておきたかったので，構築が簡単だというGitBucketを試してみた．"
slug: gitbucket
---

プライベートなGitホスティングサーバを持っておきたかったので，構築が簡単だという[GitBucket](https://github.com/takezoe/gitbucket)を試してみた．

## Installation of Tomcat

以下のコマンドで`tomcat8`をインストールする．

```sh
$ yaourt -S tomcat8
```

インストールできたら，`/etc/tomcat8/tomcat-users.xml`を以下に示すように編集する．

```diff
$ sudo diff tomcat-users.xml.bak tomcat-users.xml
32d31
< <!--
34,38c33,41
<   <role rolename="role1"/>
<   <user username="tomcat" password="tomcat" roles="tomcat"/>
<   <user username="both" password="tomcat" roles="tomcat,role1"/>
<   <user username="role1" password="tomcat" roles="role1"/>
< -->
---
>   <role rolename="manager-gui"/>
>   <role rolename="manager-script"/>
>   <role rolename="manager-jmx"/>
>   <role rolename="manager-status"/>
>   <role rolename="admin-gui"/>
>   <role rolename="admin-script"/>
>   <user username="tomcat" password="[CHANGE_ME]" roles="tomcat"/>
>   <user username="manager" password="[CHANGE_ME]" roles="manager-gui,manager-script,manager-jmx,manager-status"/>
>   <user username="admin" password="[CHANGE_ME]" roles="admin-gui"/>
```

編集できたら，以下のコマンドでTomcatを起動する．

```sh
$ sudo systemctl start tomcat8
```

起動後Tomcatを起動したサーバにsshのポート転送を用いた接続を行い，スタート画面の表示を確認する．

```sh
ssh -L 8080:localhost:8080 username@your.server.hostname
```

上記コマンドを実行し，ブラウザ上で`localhost:8080`に移動すれば良い．起動確認を済ませたら，GitBucketを動作させる下準備を行う．

## Preparation of installing GitBucket

GitBucketをダウンロードする前に，TomcatとApacheが連携して動作できるように設定を行う．まずは，`/etc/tomcat8/server.xml`の以下のような行をコメントアウトし，Tomcatへの直接接続を禁止するように変更する．

```xml
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
```

また，以下のような行が存在するかを確認し，なければ追加する．

```xml
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

さらに，以下の行を最終行付近の`</Host>`の下に追加する．

```xml
<Host name="tomcat.your.domain" appBase="myapp" unpackWARs="true" autoDeploy="true">
 <Value className="org.apache.catalina.values.AccesLogValve" directory="logs"
   prefix="tomcat.your.domain-access_log" suffix=".txt"
   pattern="%h %l %u %t &quot;%r&quot; %s %b" />
</Host>
```

追加できたら，GitBucketを配置するディレクトリの作成と/usr/share/tomcat8/へのリンク作成，GitBucketのデータを保存する.gitbucketディレクトリの作成を行う．具体的には，以下のコードを実行する．

```sh
$ sudo mkdir /var/lib/tomcat8/myapp
$ sudo chown tomcat8:tomcat8 /var/lib/tomcat8/myapp
$ sudo ln -s /var/lib/tomcat8/myapp /usr/share/tomcat8/myapp
$ sudo mkdir /usr/share/tomcat8/.gitbucket
$ sudo chown tomcat8:tomcat8 /usr/share/tomcat8/.gitbucket
```

次に，`/etc/httpd/conf/httpd.conf`の以下の2行をアンコメントする．

```conf
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
```

さらに，GitBucketにはHTTPSで接続したいため，オレオレ証明書を作成する．

```sh
$ sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key
$ sudo chmod 600 server.key
$ sudo openssl req -new -key server.key -out server.csr
$ sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

証明書が作成できたら，`/etc/httpd/conf/extra/tomcat.conf`を作成し，VirtualHostの設定を行う．

```conf
<VirtualHost *:443>
 ServerAdmin foo@foofarm.com
 ServerName tomcat.your.domain
 ErrorLog /var/log/httpd/tomcat.your.domain-error.log
 CustomLog /var/log/httpd/tomcat.your.domain-access.log common

 DocumentRoot /usr/share/tomcat8/myapp
 ProxyPass /assets !
 ProxyPass /gitbucket ajp://localhost:8009/gitbucket
 ProxyPassReverse /gitbucket ajp://localhost:8009/gitbucket
 ProxyPreserveHost on

 SSLEngine on
 SSLCertificateFile "/etc/httpd/conf/server.crt"
 SSLCertificateKeyFile "/etc/httpd/conf/server.key"
</VirtualHost>
<VirtualHost *:80>
 ServerName tomcat.your.domain
 Redirect / https://tomcat.your.domain/
</VirtualHost>
```

ファイルの作成ができたら，`/etc/httpd/conf/httpd.conf`のInclude行の最後に以下の行を追加する．

```conf
Include conf/extra/tomcat.conf
```

さて，以上でTomcatとApacheの下準備は出来た．

## Install GitBucket

いよいよGitBucketのインストールを行う．`gitbucket.war`の最新版を[GitBucketのリリースページ](https://github.com/takezoe/gitbucket/releases)からダウンロードしてこよう．

```sh
$ wget https://github.com/takezoe/gitbucket/releases/download/2.8/gitbucket.war
```

`wget`に渡しているURLは構築当時のものなので，実際に構築する際は[リリースページ](https://github.com/takezoe/gitbucket/releases)から最新版のURLを渡すこと．ダウンロードできたら，以下のコマンドで`gitbucket.war`を適切なパスに移動させる．

```sh
$ sudo mv gitbucket.war /var/lib/tomcat8/webapps/
```

移動させられたら，以下のコマンドを実行してGitBucketを動かそう．

```sh
$ sudo systemctl restart tomcat8
$ sudo systemctl enable tomcat8
$ sudo systemctl restart httpd
$ sudo systemctl enable httpd
```

正常に起動し，`/var/lib/tomcat8/myapp/gitbucket`ディレクトリが作成されていることを確認したら，`https://tomcat.your.domain/gitbucket/`にアクセスしてみよう．正常に設定できていればGitBucketのサインイン画面が出るはずだ．

