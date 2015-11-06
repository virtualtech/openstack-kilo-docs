# Ubuntu 14.04 LTS環境へのZabbix 2.2インストール手順

## パッケージのインストール
次のコマンドを実行し、ZabbixおよびZabbixの稼働に必要となるパッケージ群をインストールします。
```
$ sudo apt-get install -y php5-mysql zabbix-agent zabbix-server-mysql zabbix-java-gateway zabbix-frontend-php
```
## Zabbix用データベースの作成
次のコマンドを実行し、Zabbix用MySQLユーザおよびデータベースを作成します。
```
$ mysql -uroot -p
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';
mysql> exit
```
次のコマンドを実行し、Zabbix用データベースにテーブル等のデータベースオブジェクトを作成します。
````
$ cd /usr/share/zabbix-server-mysql/
$ zcat schema.sql.gz | mysql zabbix -uzabbix -pzabbix
$ zcat images.sql.gz | mysql zabbix -uzabbix -pzabbix
$ zcat data.sql.gz | mysql zabbix -uzabbix -pzabbix
```
## Zabbix serverの設定および起動
/etc/zabbix/zabbix_server.confを編集し、次の行を追加します。なお、MySQLユーザzabbixのパスワードを別の文字列に変更した場合は、該当文字列を指定する必要があります。
```
DBPassword=zabbix
```
/etc/default/zabbix-serverを編集し、起動可能にします。
 * 編集前
 ```
 START=no
 ```
 * 編集後
 ```
 START=yes
 ```

以上の操作を行ったのち、サービスzabbix-serverを起動します。
```
$ sudo service zabbix-server start
```
## Zabbix frontendの設定および起動
/etc/php5/apache2/php.iniを編集します。
 * 編集前
 ```
 post_max_size = 8M
 max_input_time = 60
 max_input_time = 60
 ```
 * 編集後
 ```
 post_max_size = 16M
 max_execution_time = 300
 max_input_time = 300
 date.timezone = Asia/Tokyo ←追記します
 ```

Zabbix frontendへアクセスできるよう、設定ファイルをコピーします。
```
$ sudo cp -p /usr/share/doc/zabbix-frontend-php/examples/apache.conf /etc/apache2/conf-enabled/zabbix.conf
```
これまでの設定変更を反映させるため、サービスapache2をリロードします。
```
$ sudo service apache2 reload
```
次に、Zabbix frontendの接続設定を行います。次のコマンドを実行し、一時的に権限を変更します。
```
$ sudo chmod 775 /etc/zabbix
$ sudo chgrp www-data /etc/zabbix
```
WebブラウザでZabbix frontendへアクセスします。
```
http://<Zabbix frontendのIPアドレス>/zabbix/
```
データベース接続のための設定を行います。終了後にログイン画面が表示されますので、その時点で次のコマンドを実行して権限を元に戻します。
```
$ sudo chmod 755 /etc/zabbix
$ sudo chgrp root /etc/zabbix
```
