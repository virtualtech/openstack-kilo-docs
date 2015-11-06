# CentOS 7.1環境へのHatoholインストール手順
以下の手順にしたがって操作を行うことで、CentOS 7.1上にHatohol稼働環境を構築することが可能です。なお、セキュリティ設定の変更にあたっては安全が十分に確保されるよう注意したうえで実施してください。

## インストール
1. Project Hatohol公式yumレポジトリの登録
```
# wget -P /etc/yum.repos.d/ http://project-hatohol.github.io/repo/hatohol-el7.repo
```
2. EPELレポジトリの登録
```
# rpm -ivh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
```
3. Hatoholサーバのインストール
```
# yum -y install hatohol-server
```
4. Hatohol Web Frontendのインストール
```
# yum install -y hatohol-web
```
5. 必要となる追加パッケージのインストール
```
# yum install mysql-server qpid-cpp-server
```

## セットアップ
1. /etc/my.cnf 編集
 * 編集前
```
!includedir /etc/my.cnf.d
```
 * 編集後
```
includedir /etc/my.cnf.d
```
2. /etc/my.cnf.d/server.cnf 編集  
セクション[mysqld]に、少なくとも次のパラメータを追記します。
```
[mysqld]
character-set-server = utf8
skip-character-set-client-handshake
default-storage-engine = innodb
innodb_file_per_table
```
3. MySQLの自動起動有効化と起動
```
# systemctl enable mariadb.service
# systemctl start mariadb.service
```
4. MariaDBユーザrootのパスワード変更
```
# mysqladmin password
```
5. Hatohol DBの初期化
```
hatohol-db-initiator --db_user <MySQLのrootユーザー名> --db_password <MySQLのrootパスワード>
```
初期状態で上記コマンドを実行した場合、MySQLユーザhatohol、データベースhatoholが作成されます。これらを変更する場合、事前に/etc/hatohol/hatohol.confを編集してください。
6. Hatohol Web用DBの作成
```
mysql -uroot -p
MySQL> CREATE DATABASE hatohol_client;
MySQL> GRANT ALL PRIVILEGES ON hatohol_client.* TO hatohol@localhost IDENTIFIED BY 'hatohol';
```
7. Hatohol Web用DBへのテーブル追加
```
# /usr/libexec/hatohol/client/manage.py syncdb
```
8. Hatohol serverの自動起動有効化と起動
```
# systemctl enable hatohol.service
# systemctl start hatohol.service
```
9. Hatohol Webの自動起動有効化と起動
```
# systemctl enable httpd.service
# systemctl start httpd.service
```

## セキュリティ設定の変更
CentOSインストール後の初期状態では、SElinux, firewalld, iptablesといったセキュリティ機構により他のコンピュータからのアクセスに制限が加えられます。Hatoholを使用するにあたり、これらを適切に解除する必要があります。
1. SELinuxの設定  
```
# getenforce
Enforcing
```
Enforcingの場合、次のコマンドでSElinuxポリシールールの強制適用を解除できます。
```
setenforce 0
# getenforce
Permissive
```
恒久的にSElinuxの無効化を行う場合、/etc/selinux/configを編集します。  
 * 編集前
 ```
 SELINUX=enforcing
 ```
 * 編集後
 ```
 SELINUX=permissive
 ```
 または
 ```
 SELINUX=disabled
 ```
2. firewalldの設定
firewalldの設定変更は、次のコマンドで恒久的に変更可能です。
```
# firewall-cmd --add-service=http --zone=public --permanent
```
3. iptablesの設定
```
# iptables -A IN_public_allow -p tcp -m tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
# iptables-save > /etc/sysconfig/iptables
```

## Hatohol情報の閲覧
Hatohol client (Hatohol Web)が動作しているホストのトップディレクトリをWebブラウザで表示してください。192.168.1.1でHatohol clientが動作している場合は、次のURLとなります。
```
http://192.168.1.1/
```

# Hatohol Arm Plugin Interfaceを使用する場合の操作
Hatohol Arm Plugin Interface(HAPI)を使用する場合、/etc/qpid/qpidd.confに次の行を追記します。なお、=の前後にスペースを入れてはなりません。
```
auth=no
```
