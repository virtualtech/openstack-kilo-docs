Title: OpenStack構築手順書 Kilo版
Company: 日本仮想化技術
Version:1.0.6-4

#OpenStack構築手順書 Kilo版

<div class="title">
バージョン：1.0.6-4 (2016/07/25作成)<br>
日本仮想化技術株式会社
</div>

<!-- BREAK -->

##変更履歴

|バージョン|更新日|更新内容|
|:---|:---|:---|
|0.9.0alpha1|2015/06/22|Kilo版開始|
|0.9.0alpha2|2015/07/01|編集コメントを削除、スペルミス修正、dns-nameserverを追記|
|0.9.0alpha3|2015/07/02|Cloud Archiveリポジトリーパッケージの導入方法をUbuntu公式の方法に変更|
|0.9.1|2015/07/06|openrcの内容が古かったので修正|
|0.9.2|2015/07/06|語句の統一、インデックスの修正|
|0.9.3|2015/07/06|タグなどの修正|
|0.9.4|2015/07/06|埋め込みフォントのエラーが出る問題の対処|
|0.9.5|2015/07/07|Zabbixとhatoholの手順を一部修正、注記|
|0.9.6|2015/07/08|Zabbixとhatoholの手順を一部修正|
|0.9.7|2015/07/09|監視対象の追加方法を追加|
|0.9.8|2015/07/10|誤記修正|
|0.9.9|2015/07/13|ブリッジデバイス作成時のSSH切断について注記|
|0.10.0|2015/07/15|Zabbix Agentの設定を追加|
|1.0.0|2015/08/05|プロキシー設定について追記|
|1.0.1|2015/11/04|RabbitMQの設定にhostnameを追記,ISOのURLを変更|
|1.0.2|2015/11/06|Neutronパスワードの場所表記に誤りがあったので修正|
|1.0.3|2015/11/16|本手順書の位置付けを修正|
|1.0.4|2015/11/16|表記ゆれの修正|
|1.0.5|2016/01/13|表記ゆれの修正|
|1.0.6-1|2016/04/07|5-8,7-2の書式崩れの対応|
|1.0.6-2|2016/04/08|1-1を現状に合わせて書き換え|
|1.0.6-3|2016/07/21|5-5を現状に合わせて書き換え([bug#1](https://github.com/virtualtech/openstack-kilo-docs/issues/1))|
|1.0.6-4|2016/07/25|軽微な修正|
 
<!-- BREAK -->

##目次
<!--TOC max3-->

<!-- BREAK -->

#Part.1 OpenStack 構築編
<br>
本章は、OpenStack Foundationが公開している公式ドキュメント「OpenStack Installation Guide for Ubuntu 14.04」の内容から、「Block Storage Service」までの構築手順をベースに加筆したものです。
OpenStackをUbuntu Server 14.04.2ベースで構築する手順を解説しています。
Canonical社が提供するCloud Archiveリポジトリーを使って、OpenStack Kiloを導入しましょう。

<!-- BREAK -->

## 1. 構築する環境について

### 1-1 環境構築に使用するOS

本書はCanonicalのUbuntu ServerとCloud Archiveリポジトリーのパッケージを使って、OpenStack Kiloを構築する手順を解説したものです。

OSはUbuntu Server 14.04.2 LTS(以下Ubuntu Server)のイメージを使用してインストールします。以下のURLよりイメージをダウンロードし、各サーバーへインストールします。

- <http://old-releases.ubuntu.com/releases/14.04.2/ubuntu-14.04.2-server-amd64.iso>

このドキュメントはUbuntu Server 14.04.2がリリースされた時期に執筆されたものです。
Ubuntu Server 14.04.2より新しいバージョンを使ってOpenStack Kilo環境を構築する場合は本手順で説明したよりも新しいバージョンのLinux Kernelが提供されるため、「1-6-2 カーネルの更新」の手順は不要です。

### 1-2 サーバーの構成について

本書はOpenStack環境を4台のサーバー上に構築することを想定しています。

従来の3台構成でのインストールをお望みの場合は、sqlノードでインストールおよび実行している設定、コマンドをcontrollerノードで実行してください。また設定ファイルへのデータベースの追記は、次のようにコントローラーノードを指定していただければこの手順に従って構築可能です。

(3ノード構成時のKeystoneデータベースの設定記述例)

```
connection = mysql://keystone:password@sql/keystone
↓
connection = mysql://keystone:password@controller/keystone
```

<!-- BREAK -->

### 1-3 作成するサーバー（ノード）

今回構築するOpenStack環境は、以下4台のサーバーで構成します。

+ sqlノード  
  データベースサーバー「MariaDB」の実行用のノードです。データベースはこのノード上で作成します。
+ controllerノード  
  OpenStack環境全体を管理するコントローラーとして機能します。
+ networkノード  
  外部ネットワークとインスタンスの間のネットワークを制御します。
+ computeノード  
  仮想マシンインスタンスを実行します。

### 1-4 ネットワークセグメントの設定

今回は2つのネットワークセグメントを用意し構成しています。

+ 内部ネットワーク(Instance Tunnels)
  ネットワークノードとコンピュートノード間のトンネル用に使用するネットワーク。インターネットへの接続は行えなくても構いません。

+ 外部ネットワーク(Management)
  外部との接続に使用するネットワーク。構築中はaptコマンドを使って外部リポジトリからパッケージなどをダウンロードするため、インターネット接続が必要となります。

OpenStack稼働後は、仮想マシンインスタンスに対しFloating IPアドレスを割り当てることで、外部ネットワークへ接続することができます。

なお、各種APIを外部公開する際にも使用できますが、今回の手順ではAPIの公開は行いません。

IPアドレスは以下の構成で構築されている前提で解説します。

|-|外部ネットワーク|内部ネットワーク|
|:---|:---|:---|
|インターフェース|eth0|eth1|
|ネットワーク|10.0.0.0/24|192.168.0.0/24|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

<!-- BREAK -->

### 1-5 各ノードのネットワーク設定

各ノードのネットワーク設定は以下の通りです。

+ sqlノード

|インターフェース|eth0|eth1|
|:---|:---|:---|
|IPアドレス|10.0.0.100|192.168.0.100|
|ネットマスク|255.255.255.0|255.255.255.0|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

+ controllerノード

|インターフェース|eth0|eth1|
|:---|:---|:---|
|IPアドレス|10.0.0.101|192.168.0.101|
|ネットマスク|255.255.255.0|255.255.255.0|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

+ networkノード

|インターフェース|eth0|eth1|
|:---|:---|:---|
|IPアドレス|10.0.0.102|192.168.0.102|
|ネットマスク|255.255.255.0|255.255.255.0|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

+ computeノード

|インターフェース|eth0|eth1|
|:---|:---|:---|
|IPアドレス|10.0.0.103|192.168.0.103|
|ネットマスク|255.255.255.0|255.255.255.0|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

<!-- BREAK -->

### 1-6 Ubuntu Serverのインストール

#### 1-6-1 インストール

4台のサーバーに対し、Ubuntu Serverをインストールします。要点は以下の通りです。

+ 優先ネットワークインターフェースをeth0に指定  
 + インターネットへ接続するインターフェースはeth0を使用するため、インストール中はeth0を優先ネットワークとして指定します。
+ パッケージ選択ではOpenSSH serverのみ選択
 + OSは最小インストールします。
 + computeノードではKVMを利用しますが、インストーラではVirtual machine hostのインストールを行わないでください。


【インストール時の設定パラメータ例】

|設定項目|設定例|
|:---|:---|
|初期起動時のLanguage|English|
|起動|Install Ubuntu Server|
|言語|English - English|
|地域の設定|other→Asia→Japan|
|地域の言語|United States - en_US.UTF-8|
|キーボードレイアウトの認識|No|
|キーボードの言語|Japanese→Japanese|
|優先するNIC|eth0: Ethernet|
|ホスト名|それぞれのノード名(controller, network, compute1)|

<!-- BREAK -->

|設定項目|設定例|
|:---|:---|
|ユーザ名とパスワード|フルネームで入力|
|アカウント名|ユーザ名のファーストネームで設定される|
|パスワード|任意のパスワード|
|Weak password（出ない場合も）|Yesを選択|
|ホームの暗号化|任意|
|タイムゾーン|Asia/Tokyoであることを確認|
|パーティション設定|Guided - use entire disk and set up LVM|
|パーティション選択|sdaを選択|
|パーティション書き込み|Yesを選択|
|パーティションサイズ|デフォルトのまま|
|変更の書き込み|Yesを選択|
|HTTP proxy|環境に合わせて任意|
|アップグレード|No automatic updatesを推奨|
|ソフトウェア|OpenSSH serverのみ選択|
|GRUB|Yesを選択|
|インストール完了|Continueを選択|

```
筆者注:
Ubuntuインストール時に選択した言語がインストール後も使われます。
Ubuntu Serverで日本語の言語を設定した場合、標準出力や標準エラー出力が文字化けしたり、作成されるキーペア名が文字化けするなど様々な問題が起きますので、言語は英語を設定されることを推奨します。
```

#### 1-6-2 カーネルの更新
Ubuntu Server 14.04.2はLinux Kernel 3.16系のカーネルが利用されます。次のコマンドを実行すると、Ubuntu 15.04と同等のLinux Kernel 3.19系のカーネルをUbuntu 14.04 LTSで利用することができます。Linux Kernel 3.19系ではサーバー向けに様々なパフォーマンスの改善が行われています。必要に応じてアップデートしてください。詳細は 「Ubuntu Wikiの記事(https://wiki.ubuntu.com/VividVervet/ReleaseNotes/Ja#Linux_kernel_3.19)」 をご覧ください。

```
# apt-get install -y linux-headers-generic-lts-vivid linux-image-generic-lts-vivid
```

Linux Kernel 3.16系のカーネルのアップデートが不要の場合は再起動後に実行します。

```
# apt-get remove -y linux-image-generic-lts-utopic
```

必要に応じてLinux Kernel 3.16系のカーネルを削除してください。
<!-- BREAK -->


#### 1-6-3 プロキシーの設定
外部ネットワークとの接続にプロキシーの設定が必要な場合は、aptコマンドを使ってパッケージの照会やダウンロードを行うために次のような設定をする必要があります。

- システムのプロキシー設定

```
# vi /etc/environment
http_proxy="http://proxy.example.com:8080/"
https_proxy="https://proxy.example.com:8080/"
```

- APTのプロキシー設定

```
# vi /etc/apt/apt.conf
Acquire::http::proxy "http://proxy.example.com:8080/";
Acquire::https::proxy "https://proxy.example.com:8080/";
```

より詳細な情報は下記のサイトの情報を確認ください。

- <https://help.ubuntu.com/community/AptGet/Howto>
- <http://gihyo.jp/admin/serial/01/ubuntu-recipe/0331>

<!-- BREAK -->

### 1-7 Ubuntu Serverへのログインとroot権限

Ubuntuはデフォルト設定でrootユーザーの利用を許可していないため、root権限が必要となる作業は以下のように行ってください。

+ rootユーザーで直接ログインできないので、インストール時に作成したアカウントでログインする。
+ root権限が必要な場合には、sudoコマンドを使用する。
+ rootで連続して作業したい場合には、sudo -iコマンドでシェルを起動する。

### 1-8 設定ファイル等の記述について

+ 設定ファイルは特別な記述が無い限り、必要な設定を抜粋したものです。
+ 特に変更の必要がない設定項目は省略されています。
+ [見出し]が付いている場合、その見出しから次の見出しまでの間に設定を記述します。
+ コメントアウトされていない設定項目が存在する場合には、値を変更してください。多くの設定項目は記述が存在しているため、エディタの検索機能で検索することをお勧めします。
+ 特定のホストでコマンドを実行する場合はコマンドの冒頭にホスト名を記述しています。

【設定ファイルの記述例】

```
controller# vi /etc/glance/glance-api.conf ←コマンド冒頭にこのコマンドを実行するホストを記述

[database] ←この見出しから次の見出しまでの間に以下を記述
#connection = sqlite:////var/lib/glance/glance.sqlite          ← 既存設定をコメントアウト
connection = mysql://glance:password@controller/glance         ← 追記


[keystone_authtoken] ← 見出し
#auth_host = 127.0.0.1 ← 既存設定をコメントアウト
auth_host = controller ← 追記

auth_port = 35357
auth_protocol = http
auth_uri = http://controller:5000/v2.0 ← 追記
admin_tenant_name = service ← 変更
admin_user = glance ← 変更
admin_password = password ← 変更
```
<!-- BREAK -->

## 2. OpenStackインストール事前設定

OpenStackパッケージのインストール前に各々のノードで以下の設定を行います。

+ ネットワークデバイスの設定
+ ホスト名と静的名前解決の設定
+ sysctlによるカーネルパラメータの設定
+ リポジトリーの設定とパッケージの更新
+ NTPサーバーのインストール（controllerノードのみ）
+ NTPクライアントのインストール
+ Python用MySQL/MariaDBクライアントのインストール
+ MariaDBのインストール（sqlノードのみ）
+ RabbitMQのインストール（controllerノードのみ）

### 2-1 ネットワークデバイスの設定

各ノードの/etc/network/interfacesを編集し、IPアドレスの設定を行います。

#### 2-1-1 sqlノードのIPアドレスの設定

```
sql# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.100
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
      
auto eth1
iface eth1 inet static
      address 192.168.0.100
      netmask 255.255.255.0
```

<!-- BREAK -->

#### 2-1-2 controllerノードのIPアドレスの設定

```
controller# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.101
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1

auto eth1
iface eth1 inet static
      address 192.168.0.101
      netmask 255.255.255.0
```


#### 2-1-3 networkノードのIPアドレスの設定

```
network# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.102
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1

auto eth1
iface eth1 inet static
      address 192.168.0.102
      netmask 255.255.255.0
```

<!-- BREAK -->

#### 2-1-4 computeノードのIPアドレスの設定

```
compute1# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.103
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1

auto eth1
 iface eth1 inet static
       address 192.168.0.103
       netmask 255.255.255.0
```

#### 2-1-5 ネットワーク設定反映

各ノードで変更した設定を反映させるため、ホストを再起動します。

```
# shutdown -r now
```

<!-- BREAK -->

### 2-2 ホスト名と静的名前解決の設定

各ノードの/etc/hostsに各ノードのIPアドレスとホスト名を記述し、静的名前解決の設定を行います。127.0.1.1の行はコメントアウトします。

#### 2-2-1 各ノードのホスト名の設定

各ノードのホスト名をhostnamectlコマンドを使って設定します。反映させるためには一度ログインしなおす必要があります。

（例）controllerの場合

```
# hostnamectl set-hostname controller
# cat /etc/hostname
controller
```

#### 2-2-2 各ノードの/etc/hostsの設定

すべてのノードで127.0.1.1の行をコメントアウトします。
またホスト名で名前引きできるように設定します。

（例）controllerの場合

```
# vi /etc/hosts
127.0.0.1 localhost
#127.0.1.1 controller ← 既存設定をコメントアウト
#ext
10.0.0.100 sql
10.0.0.101 controller
10.0.0.102 network
10.0.0.103 compute
#int
192.168.0.100 sql-int
192.168.0.101 controller-int
192.168.0.102 network-int
192.168.0.103 compute-int
```

<!-- BREAK -->


### 2-3 sysctlによるカーネルパラメーターの設定

Linuxのネットワークパケット処理について設定を行います。

#### 2-3-1 networkノードの/etc/sysctl.confの設定

```
network# vi /etc/sysctl.conf
net.ipv4.conf.default.rp_filter=0  ← 1から0に変更
net.ipv4.conf.all.rp_filter=0      ← 1から0に変更
net.ipv4.ip_forward=1
```

sysctlコマンドで設定を適用します。

```
network# sysctl -p
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.ip_forward = 1
```

#### 2-3-2 computeノードの/etc/sysctl.confの設定

```
compute# vi /etc/sysctl.conf
net.ipv4.conf.default.rp_filter=0      ← 1から0に変更
net.ipv4.conf.all.rp_filter=0          ← 1から0に変更
net.bridge.bridge-nf-call-iptables=1   ← 追記
net.bridge.bridge-nf-call-ip6tables=1  ← 追記
```

最近のLinux Kernelはbridgeではなくbr_netfilterモジュールを読み込む必要があるので設定します。再起動後もモジュールを読み込んでくれるように/etc/modulesに追記します。詳細は 「フォーラムの情報(http://serverfault.com/questions/697942/centos-6-elrepo-kernel-bridge-issues) 」をご覧ください。

```
compute# modprobe br_netfilter
compute# ls /proc/sys/net/bridge
bridge-nf-call-arptables  bridge-nf-filter-pppoe-tagged
bridge-nf-call-ip6tables  bridge-nf-filter-vlan-tagged
bridge-nf-call-iptables   bridge-nf-pass-vlan-input-dev

compute# echo "br_netfilter" >> /etc/modules
```

sysctlコマンドで設定を適用します。

```
compute# sysctl -p
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

<!-- BREAK -->

### 2-4 リポジトリーの設定とパッケージの更新

各ノードで以下のコマンドを実行し、Kilo向けUbuntu Cloud Archiveリポジトリを登録します。

```
# add-apt-repository cloud-archive:kilo Ubuntu Cloud Archive for OpenStack Kilo
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it    ← Enterキーを押す...
Importing ubuntu-cloud.archive.canonical.com keyring
OK
Processing ubuntu-cloud.archive.canonical.com removal keyring
OK
```

各ノードのシステムをアップデートして再起動します。

```
# apt-get update && apt-get -y dist-upgrade && reboot
```

### 2-5 NTPのインストール

各ノードで時刻を正確にするためにNTPをインストールします。

```
# apt-get install -y ntp
```

#### 2-5-1 controllerノードの/etc/ntp.confの設定

controllerノードで公開NTPサーバーと同期するNTPサーバーを構築します。
適切な公開NTPサーバー(ex.ntp.nict.jp etc..)を指定します。ネットワーク内にNTPサーバーがある場合はそのサーバーを指定します。

内容変更した場合は設定を適用するため、NTPサービスを再起動します。

```
controller# service ntp restart
```

<!-- BREAK -->

#### 2-5-2 その他ノードの/etc/ntp.confの設定

networkノードとcomputeノードでcontrollerノードと同期するNTPサーバーを構築します。

```
network# vi /etc/ntp.conf

#server 0.ubuntu.pool.ntp.org  #デフォルト設定はコメントアウトor削除
#server 1.ubuntu.pool.ntp.org
#server 2.ubuntu.pool.ntp.org
#server 3.ubuntu.pool.ntp.org
#server ntp.ubuntu.com

server controller iburst
```

設定を適用するため、NTPサービスを再起動します。

```
network# service ntp restart
```

#### 2-5-3 NTPサーバーの動作確認

構築した環境でntpq -pコマンドを実行して、各NTPサーバーが同期していることを確認します。


公開NTPサーバーと同期しているcontrollerノード

```
controller# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp-a2.nict.go. .NICT.           1 u    3   64    1    6.569   17.818   0.001
```

controllerと同期しているその他ノード

```
compute1# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*controller      ntp-a2.nict.go. 2 u  407 1024  377    1.290   -0.329   0.647
```

### 2-6 Python用MySQL/MariaDBクライアントのインストール

各ノードでPython用のMySQL/MariaDBクライアントをインストールします。

```
# apt-get install -y python-mysqldb
```

Python MySQLライブラリーはMariaDBと互換性があります。

<!-- BREAK -->

## 3. sqlノードのインストール前設定

### 3-1 MariaDBのインストール

sqlノードにデータベースサーバーのMariaDBをインストールします。

#### 3-1-1 パッケージのインストール

apt-getコマンドでmariadb-serverパッケージをインストールします。

```
sql# apt-get update
sql# apt-get install -y mariadb-server
```

インストール中にパスワードの入力を要求されますので、MariaDBのrootユーザーに対するパスワードを設定します。
本例ではパスワードとして「password」を設定します。


#### 3-1-2 MariaDB設定の変更

MariaDBの設定ファイルmy.cnfを開き以下の設定を変更します。

+ バインドアドレスをeth0に割り当てたIPアドレスへ変更
+ 文字コードをUTF-8へ変更

別のノードからMariaDBへアクセスできるようにするためバインドアドレスを変更します。加えて使用する文字コードをutf8に変更します。

※文字コードをutf8に変更しないとOpenStackモジュールとデータベース間の通信でエラーが発生します。

```
sql# vi /etc/mysql/my.cnf

[mysqld]
#bind-address = 127.0.0.1                   ← 既存設定をコメントアウト
bind-address = 10.0.0.100                   ← 追記(sqlノードのIPアドレス)
default-storage-engine = innodb             ← 追記
innodb_file_per_table                       ← 追記
collation-server = utf8_general_ci          ← 追記
init-connect = 'SET NAMES utf8'             ← 追記
character-set-server = utf8                 ← 追記
```

<!-- BREAK -->

#### 3-1-3 MariaDBサービスの再起動

変更した設定を反映させるためMariaDBのサービスを再起動します。

```
sql# service mysql restart
```

#### 3-1-4 MariaDBデータベースのセキュア化

mysql_secure_installationコマンドを実行すると、データベースのセキュリティを強化できます。必要に応じて設定を行ってください。

+ rootパスワードの入力

```
sql# mysql_secure_installation
In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.
Enter current password for root (enter for none):  password　← MariaDBのrootパスワードを入力
```

+ rootパスワードの変更

```
Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.
You already have a root password set, so you can safely answer 'n'.
Change the root password? [Y/n]  n
```

+ anonymousユーザーの削除

```
By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.
Remove anonymous users? [Y/n]  y
```

<!-- BREAK -->

+ リモートからのrootログインを禁止

本例ではデータベースの操作は全てsqlノード上で行うことを想定しているため、リモートからのrootログインは禁止に設定します。必要に応じて設定してください。

```
Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? [Y/n] y
```

+ testデーターベースの削除

```
By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.
Remove test database and access to it? [Y/n] y
```

+ 権限の再読み出し

```
Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.
Reload privilege tables now? [Y/n] y
....
Thanks for using MariaDB!
```

<!-- BREAK -->


#### 3-1-5 MariaDBクライアントのインストール

sqlノード以外のノードに、インストール済みのMariaDBと同様のバージョンのMariaDBクライアントをインストールします。

````
# apt-get update
# apt-get install -y mariadb-client-5.5 mariadb-client-core-5.5
````

#### 3-1-6 mytopのインストール

データベースの状態を確認するため、データベースパフォーマンスモニターツールのmytopをインストールします。

```
sql# apt-get update
sql# apt-get install -y mytop
```

利用するには、sqlノードで次のように実行します。ロードアベレージやデータのin/outなどの情報を確認できます。

```
sql# mytop --prompt
Password: password   　← MariaDBのrootパスワードを入力
```

<!-- BREAK -->

## 4. controllerノードのインストール前設定

### 4-1 RabbitMQのインストール

OpenStackは、オペレーションやステータス情報を各サービス間で連携するためにメッセージブローカーを使用しています。OpenStackではRabbitMQ、Qpid、ZeroMQなど複数のメッセージブローカーサービスに対応しています。
本書ではRabbitMQをインストールする例を説明します。

#### 4-1-1 パッケージのインストール

apt-getコマンドで、rabbitmq-serverパッケージをインストールします。
Cloud Archiveリポジトリーのバージョン3.4.3-2は執筆時点のバージョンでは正常に動かないので、標準リポジトリーの最新版をインストールします。

```
controller# apt-get update
controller# apt-cache policy rabbitmq-server
rabbitmq-server:
  Installed: (none)
  Candidate: 3.4.3-2~cloud0
  Version table:
     3.4.3-2~cloud0 0
        500 http://ubuntu-cloud.archive.canonical.com/ubuntu/ trusty-updates/kilo/main amd64 Packages
     3.2.4-1 0
        500 http://jp.archive.ubuntu.com/ubuntu/ trusty/main amd64 Packages
controller# apt-get install -y rabbitmq-server=3.2.4-1  ← 7/2 時点の最新版
controller# apt-mark hold rabbitmq-server               ← バージョンを固定
```

[関連バグ]

- <https://bugs.launchpad.net/cloud-archive/+bug/1449392>

<!-- BREAK -->

#### 4-1-2 openstackユーザーと権限の設定

RabbitMQにアクセスするユーザーを作成し、パーミッション権限を設定します。

```
# rabbitmqctl add_user openstack password
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 4-1-3 待ち受けIPアドレス・ポートとセキュリティ設定の変更

以下の設定ファイルを作成し、RabbitMQの待ち受けポートとIPアドレスを定義します。

+ 待ち受け設定の追加

```
controller# vi /etc/rabbitmq/rabbitmq-env.conf

RABBITMQ_NODE_IP_ADDRESS=10.0.0.101    ← controllerのIPアドレス
RABBITMQ_NODE_PORT=5672
HOSTNAME=controller
```

以下の設定ファイルを作成し、localhost以外からもRabbitMQへアクセスできるように設定します。

+ リモート認証の許可

```
controller# vi /etc/rabbitmq/rabbitmq.conf

[{rabbit, [{loopback_users, []}]}].
```

<!-- BREAK -->

#### 4-1-4 RabbitMQサービス再起動と確認

+ ログの確認

メッセージブローカーサービスが正常に動いていないと、OpenStackの各コンポーネントは正常に動きません。RabbitMQサービスの再起動と動作確認を行い、確実に動作していることを確認します。

```
controller# service rabbitmq-server restart
controller# tailf /var/log/rabbitmq/rabbit@controller.log
```

※新たなエラーが表示されなければ問題ありません。

+ デフォルトユーザーguestでRabbitMQのWeb管理画面にアクセス

次のように実行して、RabbitMQの管理画面を有効化します。

```
controller# rabbitmq-plugins enable rabbitmq_management
controller# service rabbitmq-server restart
```

ブラウザーで下記URLの管理画面にアクセスします。ユーザー:guest パスワード:guestでログインできればRabbitMQサーバー自体は正常です。

```
http://controller-node-ipaddress:15672
```

作成したopenstackユーザーでリモートからRabbitMQの管理画面にログインできないのは正常です。これはopenstackユーザーにadministrator権限が振られていないためです。ユーザー権限は「rabbitmqctl list_users」コマンドで確認、任意のユーザーに管理権限を設定するには「rabbitmqctl set_user_tags openstack administrator」のように実行するとログイン可能になります。


<!-- BREAK -->

### 4-2 環境変数設定ファイルの作成

#### 4-2-1 admin環境変数設定ファイル作成

adminユーザー用環境変数設定ファイルを作成します。

```
controller# vi ~/admin-openrc.sh

export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:35357/v3
export PS1='\u@\h \W(admin)\$ '
```

#### 4-2-2 demo環境変数設定ファイル作成

demoユーザー用環境変数設定ファイルを作成します。

```
controller# vi ~/demo-openrc.sh
 
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:5000/v3
export PS1='\u@\h \W(demo)\$ '
```

<!-- BREAK -->

## 5. Keystoneインストールと設定（controllerノード）

各サービス間の連携時に使用する認証IDサービスKeystoneのインストールと設定を行います。

### 5-1 データベースの作成・確認

Keystoneで使用するデータベースを作成します。

#### 5-1-1 データベースの作成

MariaDBにデータベースkeystoneを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

<!-- BREAK -->

#### 5-1-2 データベースの確認

sqlノードにユーザーkeystoneでログインしデータベースの閲覧が可能であることを確認します。

```
controller# mysql -h sql -u keystone -p
Enter password:  ← MariaDBのkeystoneパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)
```

#### 5-1-3 admin_tokenの決定

Keystoneのadmin_tokenに設定するトークン文字列を次のようなコマンドを実行して決定します。出力される結果はランダムな英数字になります。

```
controller# openssl rand -hex 10
64de11ce5f875e977081
```

### 5-2 パッケージのインストール

Keystoneのインストール時にサービスの自動起動が行われないようにするため、以下のように実行します。

```
controller# echo "manual" > /etc/init/keystone.override
```

apt-getコマンドでkeystoneパッケージをインストールします。

```
controller# apt-get update
controller# apt-get install -y keystone python-openstackclient apache2 libapache2-mod-wsgi memcached python-memcache
```

<!-- BREAK -->

### 5-3 設定の変更

keystoneの設定ファイルを変更します。

```
controller# vi /etc/keystone/keystone.conf

[DEFAULT]
admin_token = 64de11ce5f875e977081   ← 追記(5-1-3で出力されたキーを入力)
log_dir = /var/log/keystone          ← 設定されていることを確認
verbose = True        ← 追記(詳細なログを出力する)
...
[database]
#connection = sqlite:////var/lib/keystone/keystone.db      ← 既存設定をコメントアウト
connection = mysql://keystone:password@sql/keystone        ← 追記
...
[memcache]...servers = localhost:11211                                    ← アンコメント
...
[token]
provider = keystone.token.providers.uuid.Provider            ← アンコメント
driver = keystone.token.persistence.backends.memcache.Token  ← 追記 
...
[revoke]...driver = keystone.contrib.revoke.backends.sql.Revoke         ← アンコメント
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/keystone/keystone.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 5-4 データベースに表を作成

```
controller# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

<!-- BREAK -->

### 5-5 Apache Webサーバーの設定

+ controllerノードの/etc/apache2/apache2.confのServerNameにcontrollerノードのホスト名を設定します。

```
ServerName controller
```

+ controllerノードの/etc/apache2/sites-available/wsgi-keystone.confを作成して、次の内容を記述します。

```
Listen 5000Listen 35357<VirtualHost *:5000>  WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone display-name=%{GROUP}  WSGIProcessGroup keystone-public  WSGIScriptAlias / /var/www/cgi-bin/keystone/main  WSGIApplicationGroup %{GLOBAL}  WSGIPassAuthorization On    <IfVersion >= 2.4>       ErrorLogFormat "%{cu}t %M"    </IfVersion>  LogLevel info  ErrorLog /var/log/apache2/keystone-error.log  CustomLog /var/log/apache2/keystone-access.log combined</VirtualHost><VirtualHost *:35357>
  WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone display-name=%{GROUP}  WSGIProcessGroup keystone-admin  WSGIScriptAlias / /var/www/cgi-bin/keystone/admin  WSGIApplicationGroup %{GLOBAL}  WSGIPassAuthorization On    <IfVersion >= 2.4>      ErrorLogFormat "%{cu}t %M"    </IfVersion>  LogLevel info  ErrorLog /var/log/apache2/keystone-error.log  CustomLog /var/log/apache2/keystone-access.log combined</VirtualHost>
```

<!-- BREAK -->

+ バーチャルホストでIdentity serviceを有効に設定します。

```
controller# ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```

+ WSGIコンポーネント用のディレクトリーを作成します。

```
controller# mkdir -p /var/www/cgi-bin/keystone
```

+ WSGIコンポーネントをUpstreamリポジトリーからコピーし、先ほどのディレクトリーに展開します。

```
controller# curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=kilo-eol \| tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin
```

+ ディレクトリーとファイルのパーミッションを修正します。

```
controller# chown -R keystone:keystone /var/www/cgi-bin/keystonecontroller# chmod 755 /var/www/cgi-bin/keystone/*
```


### 5-6 サービスの再起動とDBの削除

+ Apache Webサーバーを再起動します。

```
controller# service apache2 restart
```

+ パッケージのインストール時に作成される不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/keystone/keystone.db
```

<!-- BREAK -->

### 5-7 サービスとAPIエンドポイントの作成

以下コマンドでサービスとAPIエンドポイントを設定します。

+ 環境変数の設定

```
controller# export OS_TOKEN=64de11ce5f875e977081  ← 追記(5-1-3で出力されたキーを入力)
controller# export OS_URL=http://controller:35357/v2.0
```

+ サービスを作成

```
controller# openstack service create \
  --name keystone --description "OpenStack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 492157c4ba4c432995a6ebbf579b8654 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```

+ APIエンドポイントを作成

```
controller# openstack endpoint create \--publicurl http://controller:5000/v2.0 \--internalurl http://controller:5000/v2.0 \--adminurl http://controller:35357/v2.0 \--region RegionOne identity
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://controller:35357/v2.0     |
| id           | e10594bbf242482c86e8a9076c41957e |
| internalurl  | http://controller:5000/v2.0      |
| publicurl    | http://controller:5000/v2.0      |
| region       | RegionOne                        |
| service_id   | 492157c4ba4c432995a6ebbf579b8654 |
| service_name | keystone                         |
| service_type | identity                         |
+--------------+----------------------------------+
```

<!-- BREAK -->

### 5-8 プロジェクトとユーザー、ロールの作成

以下コマンドで認証情報（テナント・ユーザー・ロール）を設定します。

+ adminプロジェクトの作成

```
controller# openstack project create --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| enabled     | True                             |
| id          | 218010a87fe5477bba7f5e25c8211614 |
| name        | admin                            |
+-------------+----------------------------------+
```

+ adminユーザーの作成

```
controller# openstack user create --password-prompt admin
User Password: password  #adminユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 9caffb5dc1d749c5b3e9493139fe8598 |
| name     | admin                            |
| username | admin                            |
+----------+----------------------------------+
```

+ adminロールの作成

```
controller# openstack role create admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9212e4ba1d07418a97fb4eaaaa275334 |
| name  | admin                            |
+-------+----------------------------------+
```

+ adminプロジェクトとユーザーにadminロールを追加

```
controller# openstack role add --project admin --user admin admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9212e4ba1d07418a97fb4eaaaa275334 |
| name  | admin                            |
+-------+----------------------------------+
```

<!-- BREAK -->

+ serviceプロジェクトを作成

```
controller# openstack project create --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| enabled     | True                             |
| id          | 5b786e6b78d248df91b8722f513e38d2 |
| name        | service                          |
+-------------+----------------------------------+
```

+ demoプロジェクトの作成

```
controller# openstack project create --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| enabled     | True                             |
| id          | 3ed2437abf474a37b305338666d9fafa |
| name        | demo                             |
+-------------+----------------------------------+
```

+ demoユーザーの作成

```
controller# openstack user create --password-prompt demo
User Password: password  #demoユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 81b6c592d5a847a1b0ee8740d14a2e3b |
| name     | demo                             |
| username | demo                             |
+----------+----------------------------------+
```

+ userロールの作成

```
controller# openstack role create user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | da8e8598734a47bd9da2404dad7b4884 |
| name  | user                             |
+-------+----------------------------------+
```

+ demoプロジェクトとdemoユーザーにuserロールを追加

```
controller# openstack role add --project demo --user demo user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | da8e8598734a47bd9da2404dad7b4884 |
| name  | user                             |
+-------+----------------------------------+
```

<!-- BREAK -->

### 5-9 Keystoneの動作の確認

他のサービスをインストールする前にIdentityサービスが正しく構築、設定されたか動作を検証します。

+ セキュリティを確保するため、一時認証トークンメカニズムを無効化します。
  + /etc/keystone/keystone-paste.iniを開き、[pipeline:public_api]と[pipeline:admin_api]と[pipeline:api_v3]セクション（訳者注:..のpipeline行）から、admin_token_authを取り除きます。

```
[pipeline:public_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension user_crud_extension public_service
...
[pipeline:admin_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension s3_extension crud_extension admin_service
...
[pipeline:api_v3]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension simple_cert_extension revoke_extension federation_extension oauth1_extension endpoint_filter_extension endpoint_policy_extension service_v3
```

+ Keystoneへの作成が完了したら環境変数をリセットします。

```
controller# unset OS_TOKEN OS_URL
```

<!-- BREAK -->


### 5-9-1 動作の確認
動作確認のためadminおよびdemoテナントに対し認証トークンを要求してみます。
admin、demoユーザーのパスワードを入力する必要があります。

+ adminユーザーとして、Identity バージョン 2.0 API から管理トークンを要求します。

```
controller# openstack --os-auth-url http://controller:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-22T10:02:36Z             |
| id         | 12ca032c6b914a7382e93d9b371b52d8 |
| project_id | 218010a87fe5477bba7f5e25c8211614 |
| user_id    | 9caffb5dc1d749c5b3e9493139fe8598 |
+------------+----------------------------------+
```

正常に応答が返ってくると、/var/log/apache2/keystone-access.logにHTTP 200と記録されます。正常に応答がない場合は/var/log/apache2/keystone-error.logを確認しましょう。

```
...
10.0.0.101 - - [23/Jun/2015:09:55:21 +0900] "GET / HTTP/1.1" 300 789 "-" "python-keystoneclient"
10.0.0.101 - - [23/Jun/2015:09:55:24 +0900] "POST /v2.0/tokens HTTP/1.1" 200 1070 "-" "python-keystoneclient"
```

+ adminユーザーとして、Identity バージョン 3.0 API から管理トークンを要求します。

```
controller# openstack --os-auth-url http://controller:35357 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name admin --os-username admin --os-auth-type password \
  token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-22T10:05:59.140858Z      |
| id         | 5bc15107d6604841b2b006d30c5b94bb |
| project_id | 218010a87fe5477bba7f5e25c8211614 |
| user_id    | 9caffb5dc1d749c5b3e9493139fe8598 |
+------------+----------------------------------+
```

<!-- BREAK -->

+ adminユーザーで管理ユーザー専用のコマンドを使って、作成したプロジェクトを表示できることを確認します。

```
controller# openstack --os-auth-url http://controller:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  project list
Password:
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 218010a87fe5477bba7f5e25c8211614 | admin   |
| 3ed2437abf474a37b305338666d9fafa | demo    |
| 5b786e6b78d248df91b8722f513e38d2 | service |
+----------------------------------+---------+
```

+ adminユーザーでユーザーを一覧表示して、先に作成したユーザーが含まれることを確認します。

```
controller# openstack --os-auth-url http://controller:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  user list
Password:
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 9caffb5dc1d749c5b3e9493139fe8598 | admin |
| 81b6c592d5a847a1b0ee8740d14a2e3b | demo  |
+----------------------------------+-------+
```

+ adminユーザーでユーザーを一覧表示して、先に作成したロールが含まれることを確認します。

```
controller# openstack --os-auth-url http://controller:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  role list
Password:
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 9212e4ba1d07418a97fb4eaaaa275334 | admin |
| da8e8598734a47bd9da2404dad7b4884 | user  |
+----------------------------------+-------+
```

<!-- BREAK -->

+ demoユーザーとして、Identity バージョン 3 API から管理トークンを要求します。

```
controller# openstack --os-auth-url http://controller:5000 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name demo --os-username demo --os-auth-type password \
  token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-22T10:17:22.735793Z      |
| id         | 658a14e8a2bd49f3aea4ab8208ca1ae5 |
| project_id | 3ed2437abf474a37b305338666d9fafa |
| user_id    | 81b6c592d5a847a1b0ee8740d14a2e3b |
+------------+----------------------------------+
```

+ demoユーザーでは管理権限が必要なコマンド、例えばユーザー一覧の表示を行おうとするとエラーになることを確認します。

```
controller# openstack --os-auth-url http://controller:5000 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name demo --os-username demo --os-auth-type password \
  user list
Password:
ERROR: openstack You are not authorized to perform the requested action: admin_required (HTTP 403)
```

<!-- BREAK -->


## 6. Glanceのインストールと設定

### 6-1 データベースの作成・確認

#### 6-1-1 データベース作成

MariaDBにデータベースglanceを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
 IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

#### 6-1-2 データベースの確認

ユーザーglanceでログインしデータベースの閲覧が可能であることを確認します。

```
controller# mysql -h sql -u glance -p
Enter password: ← MariaDBのglanceパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
+--------------------+
2 rows in set (0.00 sec)

```

<!-- BREAK -->

### 6-2 ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

admin-openrc.shを読み込むと次のように出力が変化します。

```
controller# source admin-openrc.sh
controller ~(admin)#
```

+ glanceユーザーの作成

```
controller# openstack user create --password-prompt glance
User Password: password  #glanceユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 7cd03c39bf584d49902371154b71c6fc |
| name     | glance                           |
| username | glance                           |
+----------+----------------------------------+
```

+ adminロールをglanceユーザーとserviceプロジェクトに追加

```
controller# openstack role add --project service --user glance admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9212e4ba1d07418a97fb4eaaaa275334 |
| name  | admin                            |
+-------+----------------------------------+
```

<!-- BREAK -->

+ サービスの作成

```
controller# openstack service create --name glance \--description "OpenStack Image service" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image service          |
| enabled     | True                             |
| id          | dbef33496c2e4c8aa7127077677242b9 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

+ サービスエンドポイントの作成

```
controller# openstack endpoint create \--publicurl http://controller:9292 \--internalurl http://controller:9292 \--adminurl http://controller:9292 \--region RegionOne image
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://controller:9292           |
| id           | 95206098f81d42b7b50a42e44d4f5acd |
| internalurl  | http://controller:9292           |
| publicurl    | http://controller:9292           |
| region       | RegionOne                        |
| service_id   | dbef33496c2e4c8aa7127077677242b9 |
| service_name | glance                           |
| service_type | image                            |
+--------------+----------------------------------+
```

<!-- BREAK -->


### 6-3 パッケージのインストール

apt-getコマンドでglanceとglanceクライアントパッケージをインストールします。

```
controller# apt-get update
controller# apt-get install -y glance python-glanceclient
```


### 6-4 設定の変更

Glanceの設定を行います。glance-api.conf、glance-registry.confともに、[keystone_authtoken]に追記した設定以外のパラメーターはコメントアウトします。

```
controller# vi /etc/glance/glance-api.conf

[DEFAULT]
...
verbose = True                ← 追記
...
notification_driver = noop    ← アンコメント
rpc_backend = 'rabbit'        ← アンコメント
rabbit_host = controller      ←変更
rabbit_userid = openstack     ←変更
rabbit_password = password    ←変更
...

[database]
#sqlite_db = /var/lib/glance/glance.sqlite         ← 既存設定をコメントアウト
connection = mysql://glance:password@sql/glance    ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
revocation_cache_time = 10
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = glance
password = password        ← glanceユーザーのパスワード(5-2で設定したもの)

[paste_deploy]
flavor = keystone          ← 追記

[glance_store]
default_store =file                                 ← 設定されていることを確認
filesystem_store_datadir = /var/lib/glance/images/  ← 設定されていることを確認
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-api.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

```
controller# vi /etc/glance/glance-registry.conf

[DEFAULT]
...
verbose = True                ← 追記
...
notification_driver = noop    ← アンコメント
rpc_backend = 'rabbit'        ← アンコメント
...
rabbit_host = controller      ← 変更
rabbit_userid = openstack     ← 変更
rabbit_password = password    ← 変更
...

[database]
#sqlite_db = /var/lib/glance/glance.sqlite             ← 既存設定をコメントアウト
connection = mysql://glance:password@sql/glance        ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = glancepassword = password              ← glanceユーザーのパスワード(5-2で設定したもの)

[paste_deploy]
flavor = keystone                ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-registry.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 6-5 データベースにデータ登録

下記コマンドにてglanceデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "glance-manage db_sync" glance
```

<!-- BREAK -->

### 6-6 Glanceサービスの再起動

設定を反映させるため、Glanceサービスを再起動します。

```
controller# service glance-registry restart && service glance-api restart
```

### 6-7 動作の確認と使用しないデータベースファイルの削除

サービスの再起動後、ログを参照しGlance RegistryとGlance APIサービスでエラーが起きていないことを確認します。

```
controller# tailf /var/log/glance/glance-api.log
controller# tailf /var/log/glance/glance-registry.log
```

インストール直後は作られていない場合が多いですが、コマンドを実行してglance.sqliteを削除します。

```
controller# rm /var/lib/glance/glance.sqlite
```

### 6-8 イメージの取得と登録

Glanceへインスタンス用仮想マシンイメージを登録します。ここでは、クラウド環境で主にテスト用途で利用されるLinuxディストリビューションCirrOSを登録します。

#### 6-8-1 環境変数の設定

Image serviceにAPIバージョン2.0でアクセスするため、スクリプトを修正して読み込み直します。

```
controller# cd
controller# echo "export OS_IMAGE_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.shcontroller# source admin-openrc.sh
```

#### 6-8-2 イメージ取得

CirrOSのWebサイトより仮想マシンイメージをダウンロードします。

```
controller# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

<!-- BREAK -->

#### 6-8-3 イメージ登録

ダウンロードした仮想マシンイメージをGlanceに登録します。

```
controller# glance image-create --name "cirros-0.3.4-x86_64" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare \
 --visibility public
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2015-06-23T02:50:54Z                 |
| disk_format      | qcow2                                |
| id               | 390d2978-4a97-4d27-be6e-32642f7a3789 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.4-x86_64                  |
| owner            | 218010a87fe5477bba7f5e25c8211614     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2015-06-23T02:50:54Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

#### 6-8-4 イメージ登録確認

仮想マシンイメージが正しく登録されたか確認します。

```
controller# glance image-list
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| 390d2978-4a97-4d27-be6e-32642f7a3789 | cirros-0.3.4-x86_64 |
+--------------------------------------+---------------------+
```

<!-- BREAK -->


## 7. Novaのインストールと設定（controllerノード）

### 7-1 データベースの作成・確認

#### 7-1-1 データベースの作成

MariaDBにデータベースnovaを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password:           ← MariaDBのrootパスワードpasswordを入力
```

#### 7-1-2 データベースの作成確認

※ユーザーnovaでログインしデータベースの閲覧が可能であることを確認します。

```
controller# mysql -h sql -u nova -p
Enter password: ← MariaDBのnovaパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
+--------------------+
2 rows in set (0.00 sec)

```

<!-- BREAK -->

### 7-2 ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc.sh
```

+ novaユーザーの作成

```
controller# openstack user create --password-prompt nova
User Password: password  #novaユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 186f2d77f0664d7b81b304ea6cb24660 |
| name     | nova                             |
| username | nova                             |
+----------+----------------------------------+
```

+ novaユーザーをadminロールに追加

```
controller# openstack role add --project service --user nova admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9212e4ba1d07418a97fb4eaaaa275334 |
| name  | admin                            |
+-------+----------------------------------+
```

+ novaサービスの作成

```
controller# openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | a0f7280ea95a4c1298764c9e12f99b49 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

<!-- BREAK -->

+ ComputeサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create \--publicurl http://controller:8774/v2/%\(tenant_id\)s \--internalurl http://controller:8774/v2/%\(tenant_id\)s \--adminurl http://controller:8774/v2/%\(tenant_id\)s \--region RegionOne compute
+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| adminurl     | http://controller:8774/v2/%(tenant_id)s |
| id           | 152d36c70a50474ca64deb4c2221aa5f        |
| internalurl  | http://controller:8774/v2/%(tenant_id)s |
| publicurl    | http://controller:8774/v2/%(tenant_id)s |
| region       | RegionOne                               |
| service_id   | a0f7280ea95a4c1298764c9e12f99b49        |
| service_name | nova                                    |
| service_type | compute                                 |
+--------------+-----------------------------------------+
```

<!-- BREAK -->

### 7-3 パッケージのインストール

apt-getコマンドでNova関連のパッケージをインストールします。

```
controller# apt-get update
controller# apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy \
nova-scheduler python-novaclient
```

### 7-4 設定変更

nova.confに下記の設定を追記します。

```
controller# vi /etc/nova/nova.conf

[DEFAULT]
...
rpc_backend = rabbit        ←追記
auth_strategy = keystone    ←追記

# controllerノードのIPアドレス:10.0.0.101
my_ip = 10.0.0.101                          ←追記
vncserver_listen = 10.0.0.101               ←追記
vncserver_proxyclient_address = 10.0.0.101  ←追記

(↓これ以下追記↓)
[database]
connection = mysql://nova:password@sql/nova

[oslo_messaging_rabbit]rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = novapassword = password     ← novaユーザーのパスワード(6-2で設定したもの)

[glance]
host = controller

[oslo_concurrency]lock_path = /var/lib/nova/tmp
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 7-5 データベースにデータを作成

下記コマンドにてnovaデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "nova-manage db sync" nova
```

### 7-6 Novaサービスの再起動

設定を反映させるため、Novaのサービスを再起動します。

```
controller# service nova-api restart && service nova-cert restart && \
service nova-consoleauth restart && service nova-scheduler restart && \
service nova-conductor restart && service nova-novncproxy restart
```

### 7-7 使用しないデータベースファイル削除

データベースはMariaDBを使用するため、使用しないSQLiteファイルを削除します。

```
controller# rm /var/lib/nova/nova.sqlite
```

### 7-8 Glanceとの通信確認

NovaのコマンドラインインターフェースでGlanceと通信してGlanceと相互に通信できているかを確認します。

```
controller# nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| 390d2978-4a97-4d27-be6e-32642f7a3789 | cirros-0.3.4-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```

※Glanceに登録したCirrOSイメージが表示できていれば問題ありません。

<!-- BREAK -->


## 8. Nova-Computeのインストール・設定（computeノード）

### 8-1 パッケージのインストール

```
compute# apt-get update
compute# apt-get install -y nova-compute sysfsutils
```

### 8-2 設定の変更

novaの設定ファイルを変更します。

```
compute# vi /etc/nova/nova.conf

[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone

my_ip = 10.0.0.103  ← IPアドレスで指定
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 10.0.0.103  ← IPアドレスで指定
novncproxy_base_url = http://controller:6080/vnc_auto.html
vnc_keymap = ja                            ← 日本語キーボードの設定

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = novapassword = password     ← novaユーザーのパスワード(6-2で設定したもの)

[glance]
host = controller

[oslo_concurrency]lock_path = /var/lib/nova/tmp
```


次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

nova-computeの設定ファイルを開き、KVMを利用するように設定変更します。「egrep -c '(vmx|svm)' /proc/cpuinfo」とコマンドを実行して、0と出たらqemu、0以上の数字が出たらkvmをvirt_typeパラメーターに設定する必要があります。

まず次のようにコマンドを実行し、KVMが動く環境であることを確認します。CPUがVMXもしくはSVM対応であるか、コア数がいくつかを出力しています。0と表示される場合は後述の設定でvirt_type = qemuを設定します。

```
# cat /proc/cpuinfo |egrep 'vmx|svm'|wc -l
4
```

VMXもしくはSVM対応CPUの場合はvirt_type = kvmと設定することにより、仮想化部分のパフォーマンスが向上します。

```
compute# vi /etc/nova/nova-compute.conf

[libvirt]...
virt_type = kvm
```

<!-- BREAK -->

### 8-3 Nova-Computeサービスの再起動

設定を反映させるため、Nova-Computeのサービスを再起動します。

```
compute# service nova-compute restart
```

### 8-4 controllerノードとの疎通確認

疎通確認はcontrollerノード上にて、admin環境変数設定ファイルを読み込んで行います。

```
controller# source admin-openrc.sh
```

#### 8-4-1 ホストリストの確認

controllerノードとcomputeノードが相互に接続できているか確認します。もし、StateがXXXなサービスがあった場合は、該当のサービスをserviceコマンドで起動してください。

```
controller# date -u
Wed Jul  1 08:59:20 UTC 2015           ← 現在時刻を確認

controller# nova-manage service list   ← Novaサービスステータスを確認
No handlers could be found for logger "oslo_config.cfg"
Binary           Host          Zone          Status     State Updated_At
nova-cert        controller    internal      enabled    :-)   2015-07-01 08:59:17
nova-consoleauth controller    internal      enabled    :-)   2015-07-01 08:59:17
nova-scheduler   controller    internal      enabled    :-)   2015-07-01 08:59:17
nova-conductor   controller    internal      enabled    :-)   2015-07-01 08:59:16
nova-compute     compute       nova          enabled    :-)   2015-07-01 08:59:18
```
※一覧にcomputeが表示されていれば問題ありません。

#### 8-4-2 ハイパーバイザの確認

controllerノードよりcomputeノードのハイパーバイザが取得可能か確認します。

```
controller# nova hypervisor-list
+----+---------------------+-------+---------+
| ID | Hypervisor hostname | State | Status  |
+----+---------------------+-------+---------+
| 1  | compute             | up    | enabled |
+----+---------------------+-------+---------+
```

※Hypervisor hostname一覧にcomputeが表示されていれば問題ありません。

<!-- BREAK -->


## 9. Neutronのインストール・設定（controllerノード）


### 9-1 データベース作成・確認

#### 9-1-1 データベースの作成

MariaDBにデータベースneutronを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

#### 9-1-2 データベースの確認

MariaDBにNeutronのデータベースが登録されたか確認します。

```
controller# mysql -h sql -u neutron -p
Enter password: ← MariaDBのneutronパスワードpasswordを入力
...

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
+--------------------+
2 rows in set (0.00 sec)

```

※ユーザーneutronでログイン可能でデータベースが閲覧可能なら問題ありません。

<!-- BREAK -->

### 9-2 ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc.sh
```

+ neutronユーザーの作成

```
controller# openstack user create --password-prompt neutron
User Password: password  #neutronユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | df3e33244d6548108edeaa7cc7b1789f |
| name     | neutron                          |
| username | neutron                          |
+----------+----------------------------------+
```

+ neutronユーザーをadminロールに追加

```
controller# openstack role add --project service --user neutron admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9212e4ba1d07418a97fb4eaaaa275334 |
| name  | admin                            |
+-------+----------------------------------+
```

<!-- BREAK -->

+ neutronサービスの作成

```
controller# openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | efbab190a3b64d9db44725c3dc99b4c3 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

+ neutronサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create \--publicurl http://controller:9696 \--adminurl http://controller:9696 \--internalurl http://controller:9696 \--region RegionOne network
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://controller:9696           |
| id           | fe4c5a9f82574cbeb2f17685bb0956c2 |
| internalurl  | http://controller:9696           |
| publicurl    | http://controller:9696           |
| region       | RegionOne                        |
| service_id   | efbab190a3b64d9db44725c3dc99b4c3 |
| service_name | neutron                          |
| service_type | network                          |
+--------------+----------------------------------+
```

<!-- BREAK -->


### 9-3 パッケージのインストール

```
controller# apt-get update
controller# apt-get install -y neutron-server neutron-plugin-ml2 python-neutronclient
```

### 9-4 設定の変更

+ Neutron Serverの設定

```
controller# vi /etc/neutron/neutron.conf 

[DEFAULT]...
verbose = Truerpc_backend = rabbit          ←アンコメント
auth_strategy = keystone      ←アンコメント

core_plugin = ml2             ←確認service_plugins = router      ←追記allow_overlapping_ips = True  ←変更

notify_nova_on_port_status_changes = True   ←アンコメントnotify_nova_on_port_data_changes = True     ←アンコメントnova_url = http://controller:8774/v2        ←変更

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = neutronpassword = password       ← neutronユーザーのパスワード(9-2で設定したもの)

[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite    ← 既存設定をコメントアウト
connection = mysql://neutron:password@sql/neutron          ← 追記

[nova]（以下末尾に追記）
...
auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultregion_name = RegionOneproject_name = serviceusername = novapassword = password     ← novaユーザーのパスワード(6-2で設定したもの)

（次ページに続きます...）
```

<!-- BREAK -->

```
（前ページ/etc/neutron/neutron.confの続き）
[oslo_messaging_rabbit]（以下追記）...
# Deprecated group/name - [DEFAULT]/fake_rabbit
# fake_rabbit = falserabbit_host = controller
rabbit_userid = openstack
rabbit_password = password
```

[keystone_authtoken]セクションは追記した設定以外は取り除くかコメントアウトしてください。

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```



+ ML2プラグインの設定

```
controller# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
...
type_drivers = flat,vlan,gre,vxlan       ← 追記
tenant_network_types = gre               ← 追記
mechanism_drivers = openvswitch          ← 追記

[ml2_type_gre]
...
tunnel_id_ranges = 1:1000                ← 追記

[securitygroup]
...
enable_security_group = True             ← アンコメント                                                      
enable_ipset = True                      ← アンコメント
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver   ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/ml2_conf.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 9-5 設定の変更

Novaの設定ファイルにNeutronの設定を追記します。

```
controller# vi /etc/nova/nova.conf

[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[neutron]
url = http://controller:9696
auth_strategy = keystone
admin_auth_url = http://controller:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = password       ← neutronユーザーのパスワード(9-2で設定したもの)
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 9-6 データベースの作成

コマンドを実行して、エラーがでないで完了することを確認します。

```
controller# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
INFO  [alembic.migration] Context impl MySQLImpl.
INFO  [alembic.migration] Will assume non-transactional DDL.
...
INFO  [alembic.migration] Running upgrade 28a09af858a8 -> 20c469a5f920, add index for port
INFO  [alembic.migration] Running upgrade 20c469a5f920 -> kilo, kilo
```

<!-- BREAK -->

### 9-7 ログの確認

インストールしたNeutron Serverのログを参照し、エラーが出ていないことを確認します。

```
controller# tailf /var/log/neutron/neutron-server.log
...
2015-07-03 11:32:50.570 8809 INFO neutron.service [-] Neutron service started, listening on 0.0.0.0:9696
2015-07-03 11:32:50.571 8809 INFO oslo_messaging._drivers.impl_rabbit [-] Connecting to AMQP server on controller:5672
2015-07-03 11:32:50.585 8809 INFO neutron.wsgi [-] (8809) wsgi starting up on http://0.0.0.0:9696/
2015-07-03 11:32:50.592 8809 INFO oslo_messaging._drivers.impl_rabbit [-] Connected to AMQP server on controller:5672
```


### 9-8 使用しないデータベースファイル削除

```
controller# rm /var/lib/neutron/neutron.sqlite
```

### 9-9 controllerノードのNeutronと関連サービスの再起動

設定を反映させるため、controllerノードの関連サービスを再起動します。

```
controller# service nova-api restart && service neutron-server restart
```

<!-- BREAK -->

### 9-10 動作の確認

Neutron Serverの動作を確認するため、拡張機能一覧を表示するneutronコマンドを実行します。

```
controller:~# source admin-openrc.sh
controller:~# neutron ext-list
+-----------------------+-----------------------------------------------+
| alias                 | name                                          |
+-----------------------+-----------------------------------------------+
| security-group        | security-group                                |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| net-mtu               | Network MTU                                   |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| provider              | Provider Network                              |
| agent                 | agent                                         |
| quotas                | Quota management support                      |
| subnet_allocation     | Subnet Allocation                             |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| l3-ha                 | HA Router extension                           |
| multi-provider        | Multi Provider Network                        |
| external-net          | Neutron external network                      |
| router                | Neutron L3 Router                             |
| allowed-address-pairs | Allowed Address Pairs                         |
| extraroute            | Neutron Extra Route                           |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
```

<!-- BREAK -->


## 10. Neutronのインストール・設定（networkノード）

### 10-1 パッケージのインストール

```
network# apt-get update
network# apt-get install -y neutron-plugin-ml2 neutron-plugin-openvswitch-agent \neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

### 10-2 設定の変更

+ Neutronの設定

```
network# vi /etc/neutron/neutron.conf

[DEFAULT]
...
verbose = True                    ← 変更
rpc_backend = rabbit              ← コメントアウトをはずす
auth_strategy = keystone          ← コメントアウトをはずす

core_plugin = ml2                 ← 確認
service_plugins = router          ← 追記
allow_overlapping_ips = True      ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = neutronpassword = password       ← neutronユーザーのパスワード(9-2で設定したもの)

[database]
# This line MUST be changed to actually run the plugin.
# Example:
#connection = sqlite:////var/lib/neutron/neutron.sqlite  ←コメントアウト

[oslo_messaging_rabbit]...
# fake_rabbit = falserabbit_host = controllerrabbit_userid = openstackrabbit_password = password
```

本書の構成では、ネットワークノードのNeutron.confにはデータベースの指定は不要です。

次のコマンドを実行して正しく設定を行ったか確認します。

```
network# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ ML2 Plug-inの設定

```
network# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
...
type_drivers = flat,vlan,gre,vxlan       ← 追記
tenant_network_types = gre               ← 追記
mechanism_drivers = openvswitch          ← 追記

[ml2_type_flat]
...
flat_networks = external                 ← 追記

[ml2_type_gre]
...
tunnel_id_ranges = 1:1000                ← 追記

[securitygroup]
...
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[agent]
tunnel_types = gre                       ← 追記

[ovs]
local_ip = 192.168.0.102                 ← 追記(networkノードのInternal側)
enable_tunneling = True                  ← 追記
bridge_mappings = external:br-ex         ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
network# less /etc/neutron/plugins/ml2/ml2_conf.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ Layer-3 (L3) agentの設定

```
network# vi /etc/neutron/l3_agent.ini

[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver    ← アンコメント
router_delete_namespaces = True     ← 変更
external_network_bridge = br-ex     ← アンコメント
verbose = True                      ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
network# less /etc/neutron/l3_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

+ DHCP agentの設定

```
network# vi /etc/neutron/dhcp_agent.ini

[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver    ← アンコメント
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq                         ← アンコメント
dhcp_delete_namespaces = True                                          ← 変更
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf    ← 追記
verbose = True    ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
network# less /etc/neutron/dhcp_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

+ DHCPオプションでMTUの設定

dnsmasq-neutron.confファイルを新規作成して、DHCPオプションを設定します。

```
network# vi /etc/neutron/dnsmasq-neutron.conf
dhcp-option-force=26,1454
```

<!-- BREAK -->

+ Metadata agentの設定

```
network# vi /etc/neutron/metadata_agent.ini

[DEFAULT]
...
auth_url = http://localhost:5000/v2.0       ← コメントアウト
admin_tenant_name = %SERVICE_TENANT_NAME%   ← コメントアウト
admin_user = %SERVICE_USER%                 ← コメントアウト
admin_password = %SERVICE_PASSWORD%         ← コメントアウト
auth_region = RegionOne                     ← 確認
（以下追記）
verbose = True
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = neutronpassword = password       ← neutronユーザーのパスワード(9-2で設定したもの)
nova_metadata_ip = controller
metadata_proxy_shared_secret = password
```

metadata_proxy_shared_secretはコマンドを実行して生成したハッシュ値を設定することを推奨します。

[実行例]

```
# openssl rand -hex 10
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
network# less /etc/neutron/metadata_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 10-3 設定の変更

controllerノードのNovaの設定ファイルに追記します。

```
controller# vi /etc/nova/nova.conf

[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = password  ← ハッシュ値(9-2「Metadata agent」に設定したものと同じもの)
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

controllerノードのnova-apiサービスを再起動します。

```
controller# service nova-api restart
```

<!-- BREAK -->

### 10-4 networkノードのOpen vSwitchサービスの再起動

OpenStackのネットワークサービス設定を反映させるため、networkノードでOpen vSwitchのサービスを再起動します。

```
network# service openvswitch-switch restart
```

### 10-5 ブリッジデバイス設定

内部通信用と外部通信用のブリッジを作成して外部通信用ブリッジに共有ネットワークデバイスを接続します。

```
network# ovs-vsctl add-br br-ex ; ovs-vsctl add-port br-ex eth0
```

__注意:__
このコマンドを実行するとnetworkノードへのSSH接続が切断されます。SSH接続ではなく、
HP iLo、Dell iDracなどのリモートコンソールやサーバーコンソール上でコマンドを実行することを推奨します。


### 10-6 サービスの再起動

設定を反映するために、関連サービスを再起動します。

```
network# service neutron-plugin-openvswitch-agent restartnetwork# service neutron-l3-agent restartnetwork# service neutron-dhcp-agent restartnetwork# service neutron-metadata-agent restart
```

<!-- BREAK -->

### 10-7 ブリッジデバイス設定確認

ブリッジの作成・設定を確認します。

#### 10-7-1 ブリッジの確認

```
network# ovs-vsctl list-br
br-ex
br-int
br-tun
```

※add-brしたブリッジが表示されていれば問題ありません。


#### 10-7-2 外部接続用ブリッジと共有ネットワークデバイスの接続確認

```
network# ovs-vsctl list-ports br-ex
eth0
phy-br-ex
```

※add-port で設定したネットワークデバイスが表示されていれば問題ありません。

<!-- BREAK -->

### 10-8 ネットワークインタフェースの設定変更

Management側に接続されたNICを使って、仮想NIC(br-ex)を作成します。

```
network# vi /etc/network/interfaces

# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual                        ← 既存設定を変更
        up ip link set dev $IFACE up          ← 既存設定を変更
        down ip link set dev $IFACE down      ← 既存設定を変更
        
auto br-ex                                        ← 追記
iface br-ex inet static                           ← 追記
        address 10.0.0.102                        ← 追記
        netmask 255.255.255.0                     ← 追記
        gateway 10.0.0.1                          ← 追記
        dns-nameservers 10.0.0.1                  ← 追記

auto eth1
iface eth1 inet static
        address 192.168.0.102
        netmask 255.255.255.0
```

### 10-9 networkノードの再起動

インタフェース設定を適用するために、システムを再起動します。

```
network# reboot
```

<!-- BREAK -->

### 10-10 ブリッジの設定を確認

各種ブリッジが正常に設定されていることを確認します。

```
network# ip a |grep 'LOOPBACK\|BROADCAST\|inet'
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master ovs-system state UP group default qlen 1000
    inet6 fe80::20c:29ff:fe61:9417/64 scope link
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.14.102/24 brd 192.168.14.255 scope global eth1
    inet6 fe80::20c:29ff:fe61:9421/64 scope link
4: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default
5: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    inet 172.17.14.102/24 brd 172.17.14.255 scope global br-ex
    inet6 fe80::20c:29ff:fe61:9417/64 scope link
6: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default
7: br-tun: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default
```

### 10-11 Neutronサービスの動作確認

構築したNeutronのエージェントが正しく認識され、稼働していることを確認します。

```
controller# source admin-openrc.sh
controller# neutron agent-list -c host -c alive -c binary
+---------+-------+---------------------------+
| host    | alive | binary                    |
+---------+-------+---------------------------+
| network | :-)   | neutron-dhcp-agent        |
| network | :-)   | neutron-l3-agent          |
| network | :-)   | neutron-metadata-agent    |
| network | :-)   | neutron-openvswitch-agent |
+---------+-------+---------------------------+
```

<!-- BREAK -->


## 11. Neutronのインストール・設定（computeノード）

### 11-1 パッケージのインストール

```
compute# apt-get update
compute# apt-get install -y neutron-plugin-ml2 neutron-plugin-openvswitch-agent
```

### 11-2 設定の変更

+ Neutronの設定

```
compute# vi /etc/neutron/neutron.conf

[DEFAULT]
...
verbose = True
rpc_backend = rabbit                  ← アンコメント
auth_strategy = keystone              ← アンコメント

core_plugin = ml2                     ← 確認
service_plugins = router              ← 追記
allow_overlapping_ips = True          ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = neutronpassword = password       ← neutronユーザーのパスワード(9-2で設定したもの)

[database]
# This line MUST be changed to actually run the plugin.
# Example:
# connection = sqlite:////var/lib/neutron/neutron.sqlite   ← コメントアウト

[oslo_messaging_rabbit]
...
# fake_rabbit = false
rabbit_host = controller           ← 追記
rabbit_userid = openstack          ← 追記
rabbit_password = password         ← 追記
```

本書の構成では、コンピュートノードのNeutron.confにはデータベースの指定は不要です。

次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ ML2 Plug-inの設定

```
compute# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat,vlan,gre,vxlan  ← 追記
tenant_network_types = gre          ← 追記
mechanism_drivers = openvswitch     ← 追記

[ml2_type_gre]
tunnel_id_ranges = 1:1000           ← 追記

[securitygroup]
enable_security_group = True        ← 追記
enable_ipset = True                 ← 追記
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver     ← 追記

[ovs]                               ← 追記
local_ip = 192.168.0.103            ← 追記(computeノードのInternal側)
enable_tunneling = True             ← 追記

[agent]                             ← 追記
tunnel_types = gre                  ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/neutron/plugins/ml2/ml2_conf.ini | grep -v "^\s*$" | grep -v "^\s*#"
```


### 11-3 computeノードのOpen vSwitchサービスの再起動

設定を反映させるため、computeノードのOpen vSwitchのサービスを再起動します。

```
compute# service openvswitch-switch restart
```

<!-- BREAK -->

### 11-4 computeノードのネットワーク設定

デフォルトではComputeはレガシーなネットワークを利用します。Neutronを利用するように設定を変更します。

```
compute# vi /etc/nova/nova.conf

[DEFAULT]...network_api_class = nova.network.neutronv2.api.APIsecurity_group_api = neutronlinuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriverfirewall_driver = nova.virt.firewall.NoopFirewallDriver

[neutron]url = http://controller:9696auth_strategy = keystoneadmin_auth_url = http://controller:35357/v2.0
admin_tenant_name = serviceadmin_username = neutronadmin_password = password       ← neutronユーザーのパスワード(9-2で設定したもの)
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 11-5 computeノードのNeutronと関連サービスを再起動

ネットワーク設定を反映させるため、compute1ノードのNeutronと関連のサービスを再起動します。

```
compute# service nova-compute restart && service neutron-plugin-openvswitch-agent restart
```

### 11-6 ログの確認

```
compute# grep "ERROR\|WARNING" /var/log/neutron/*
```

 ※何も表示されなければ問題ありません。RabbitMQの接続確立に時間がかかり、その間「AMQP server on 127.0.0.1:5672 is unreachable」というエラーが出力される場合があります。

### 11-7 Neutronサービスの動作の確認

構築したNeutronのエージェントが正しく認識され、稼働していることを確認します。

```
controller# source admin-openrc.sh
controller# neutron agent-list -c host -c alive -c binary
+---------+-------+---------------------------+
| host    | alive | binary                    |
+---------+-------+---------------------------+
| network | :-)   | neutron-dhcp-agent        |
| network | :-)   | neutron-l3-agent          |
| network | :-)   | neutron-metadata-agent    |
| compute | :-)   | neutron-openvswitch-agent | ← 追加された出力
| network | :-)   | neutron-openvswitch-agent |
+---------+-------+---------------------------+
```

 ※コンピュートが追加され、正常に稼働していることが確認できれば問題ありません。

<!-- BREAK -->


## 12. 仮想ネットワーク設定（controllerノード）

### 12-1 外部接続ネットワークの設定

#### 12-1-1 admin環境変数ファイルの読み込み

外部接続用ネットワーク作成するためにadmin環境変数を読み込みます。

```
controller# source admin-openrc.sh
```

#### 12-1-2 外部ネットワーク作成

ext-netという名前で外部用ネットワークを作成します。

```
controller# neutron net-create ext-net --router:external \--provider:physical_network external --provider:network_type flat
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 37bc47f1-e8b9-4d43-ab3b-c1926b2b42b3 |
| mtu                       | 0                                    |
| name                      | ext-net                              |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 218010a87fe5477bba7f5e25c8211614     |
+---------------------------+--------------------------------------+
```

<!-- BREAK -->

#### 12-1-3 外部ネットワーク用サブネットを作成

ext-subnetという名前で外部ネットワーク用サブネットを作成します。

```
controller# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=10.0.0.200,end=10.0.0.250 \
  --disable-dhcp --gateway 10.0.0.1 10.0.0.0/24
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "10.0.0.200", "end": "10.0.0.250"}       |
| cidr              | 10.0.0.0/24                                        |
| dns_nameservers   |                                                    |
| enable_dhcp       | False                                              |
| gateway_ip        | 10.0.0.1                                           |
| host_routes       |                                                    |
| id                | 406b93cb-3f55-49b3-8ce4-74259ac00526               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | ext-subnet                                         |
| network_id        | daf2a1a8-615d-4105-bfb4-60a8380350ef               |
| subnetpool_id     |                                                    |
| tenant_id         | 218010a87fe5477bba7f5e25c8211614                   |
+-------------------+----------------------------------------------------+
```

### 12-2 インスタンス用ネットワークの設定

#### 12-2-1 demo環境変数ファイルの読み込み

インスタンス用ネットワーク作成するためにdemo環境変数読み込みます。

```
controller# source demo-openrc.sh
```

<!-- BREAK -->

#### 12-2-2 インスタンス用ネットワークの作成

demo-netという名前でインスタンス用ネットワークを作成します。

```
controller# neutron net-create demo-net
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | de074cba-bcad-47ca-ab6d-058a974000b5 |
| mtu             | 0                                    |
| name            | demo-net                             |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 3ed2437abf474a37b305338666d9fafa     |
+-----------------+--------------------------------------+
```

#### 11-2-3 インスタンス用ネットワークサブネットを作成

demo-subnetという名前でインスタンス用ネットワークサブネットを作成します。

```
controller# neutron subnet-create demo-net 192.168.0.0/24 \--name demo-subnet --gateway 192.168.0.1 --dns-nameserver 8.8.8.8
Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| cidr              | 192.168.0.0/24                                   |
| dns_nameservers   |                                                  |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.0.1                                      |
| host_routes       |                                                  |
| id                | 92b87319-b287-4bcc-988c-003215c1580a             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | demo-subnet                                      |
| network_id        | 6be6a7ef-ef68-4c84-9b3f-0a6e41aee52b             |
| subnetpool_id     |                                                  |
| tenant_id         | 3ed2437abf474a37b305338666d9fafa                 |
+-------------------+--------------------------------------------------+
```

<!-- BREAK -->

### 12-3 仮想ネットワークルーターの設定

仮想ネットワークルーターを作成して外部接続用ネットワークとインスタンス用ネットワークをルーターに接続し、双方でデータのやり取りを行えるようにします。


#### 12-3-1 demo-routerを作成

仮想ネットワークルータを作成します。

```
controller# neutron router-create demo-router
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 8dea222a-cf31-4de0-a946-891026c21364 |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 3ed2437abf474a37b305338666d9fafa     |
+-----------------------+--------------------------------------+
```

#### 12-3-2 demo-routerにサブネットを追加

仮想ネットワークルーターにインスタンス用ネットワークを接続します。

```
controller# neutron router-interface-add demo-router demo-subnet
Added interface a66a184a-55b3-49d8-bbbf-3bbf2fe32de2 to router demo-router.
```

#### 12-3-3 demo-routerにゲートウェイを追加

仮想ネットワークルーターに外部ネットワークを接続します。

```
controller# neutron router-gateway-set demo-router ext-net
Set gateway for router demo-router
```

<!-- BREAK -->

## 13. 仮想ネットワーク設定確認（networkノード）

### 13-1 仮想ネットワークルーターの確認

以下コマンドで仮想ネットワークルーターが作成されているか確認します。

```
network# ip netns
qdhcp-ed07c38c-8609-43d8-ae02-582f9f202a3e
qrouter-7c1ca8eb-eaa0-4a68-843d-daca30824693
```

※qrouter~~ という名前の行が表示されていれば問題ありません。

### 13-2 仮想ルーターのネームスペースのIPアドレスを確認

仮想ルーターと外部用ネットワークの接続を確認します。

```
network# ip netns exec `ip netns | grep qrouter` ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
13: qr-65249869-77: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether fa:16:3e:d0:df:2c brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/24 brd 192.168.0.255 scope global qr-65249869-77
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fed0:df2c/64 scope link
       valid_lft forever preferred_lft forever
14: qg-bd7c5797-3f: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether fa:16:3e:35:80:8f brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.200/24 brd 10.0.0.255 scope global qg-bd7c5797-3f
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe35:808f/64 scope link
       valid_lft forever preferred_lft forever
```

※外部アドレス（この環境では10.0.0.0/24）のアドレスを確認します。

<!-- BREAK -->

### 13-3 仮想ゲートウェイの疎通を確認

仮想ルーターの外から仮想ルーターに対して疎通可能かを確認します。

```
network# ping 10.0.0.200
PING 10.0.0.200 (10.0.0.200) 56(84) bytes of data.
64 bytes from 10.0.0.200: icmp_seq=1 ttl=64 time=1.17 ms
64 bytes from 10.0.0.200: icmp_seq=2 ttl=64 time=0.074 ms
64 bytes from 10.0.0.200: icmp_seq=3 ttl=64 time=0.061 ms
64 bytes from 10.0.0.200: icmp_seq=4 ttl=64 time=0.076 ms
```

仮想ルーターからゲートウェイ、外部ネットワークにアクセス可能か確認します。

```
network# ip netns exec `ip netns | grep qrouter` ping 10.0.0.1
network# ip netns exec `ip netns | grep qrouter` ping virtualtech.jp
```

※応答が返ってくれば問題ありません。各ノードからPingコマンドによる疎通確認を実行しましょう。

### 13-4 インスタンスの起動確認

controller、network、computeノードの最低限の構成が出来上がってので、ここでOpenStack環境がうまく動作しているか確認しましょう。
まずはコマンドを使ってインスタンスを起動するために必要な情報を集める所から始めます。環境設定ファイルを読み込んで、各コマンドを実行し、情報を集めてください。

```
controller# source demo-openrc.sh

controller# glance image-list
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| 572104d5-a901-4432-be41-17e37901a5f7 | cirros-0.3.4-x86_64 |
+--------------------------------------+---------------------+

controller# neutron net-list -c name -c id
+----------+--------------------------------------+
| name     | id                                   |
+----------+--------------------------------------+
| demo-net | 858c0fe5-ea00-4426-aa4e-f2a2484b2471 |
| ext-net  | e0cbae79-2973-4870-93de-09dfbd3e76e4 |
+----------+--------------------------------------+

controller# nova secgroup-list
+--------------------------------------+---------+------------------------+
| Id                                   | Name    | Description            |
+--------------------------------------+---------+------------------------+
| a66e2962-312f-45a4-bfd3-f86ec69c5582 | default | Default security group |
+--------------------------------------+---------+------------------------+

controller# nova flavor-list
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

nova bootコマンドを使って、インスタンスを起動します。正常に起動したらnova deleteコマンドでインスタンスを削除してください。

```
controller# nova boot --flavor m1.tiny --image "cirros-0.3.4-x86_64" --nic net-id=858c0fe5-ea00-4426-aa4e-f2a2484b2471 --security-group a66e2962-312f-45a4-bfd3-f86ec69c5582 vm1
(インスタンスを起動)

controller# watch nova list
(インスタンス一覧を表示)
+--------------------------------------+------+--------+------------+-------------+----------------------+
| ID                                   | Name | Status | Task State | Power State | Networks             |
+--------------------------------------+------+--------+------------+-------------+----------------------+
| 5eddf2a7-0287-46ef-b656-18ff51f1c605 | vm1  | ACTIVE | -          | Running     | demo-net=192.168.0.6 |
+--------------------------------------+------+--------+------------+-------------+----------------------+

# grep "ERROR\|WARNING" /var/log/rabbitmq/*.log
# grep "ERROR\|WARNING" /var/log/openvswitch/*
# grep "ERROR\|WARNING" /var/log/neutron/*
# grep "ERROR\|WARNING" /var/log/nova/*
(各ノードの関連サービスでエラーが出ていないことを確認)

controller# nova delete vm1
Request to delete server vm1 has been accepted.
(起動したインスタンスを削除)
```

<!-- BREAK -->


## 14. Cinderのインストール（controllerノード）

### 14-1 データベース作成・確認

#### 14-1-1 データベースの作成

MariaDBのデータベースにCinderのデータベースを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

#### 14-1-2 データベースの確認

MariaDBにCinderのデータベースが登録されたか確認します。

```
controller# mysql -h sql -u cinder -p
Enter password: ← MariaDBのcinderパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| cinder             |
+--------------------+
2 rows in set (0.00 sec)

```

※ユーザーcinderでログイン可能でデータベースの閲覧が可能なら問題ありません。

<!-- BREAK -->


### 14-2 ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc.sh
```

+ cinderユーザーの作成

```
controller# openstack user create --password-prompt cinder
User Password: password  #cinderユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | f63d06a517eb484f919276bdba5b9567 |
| name     | cinder                           |
| username | cinder                           |
+----------+----------------------------------+
```

+ cinderユーザーをadminロールに追加

```
controller# openstack role add --project service --user cinder admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9212e4ba1d07418a97fb4eaaaa275334 |
| name  | admin                            |
+-------+----------------------------------+
```

<!-- BREAK -->

+ cinderサービスの作成

```
controller# openstack service create --name cinder \--description "OpenStack Block Storage" volume
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 546be85aaa664c71bc35add65c98a224 |
| name        | cinder                           |
| type        | volume                           |
+-------------+----------------------------------+

controller# openstack service create --name cinderv2 \--description "OpenStack Block Storage" volumev2
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 77ac9832fdcd485b928c118327027757 |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+
```

<!-- BREAK -->

+ Block StorageサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create \--publicurl http://controller:8776/v2/%\(tenant_id\)s \--internalurl http://controller:8776/v2/%\(tenant_id\)s \--adminurl http://controller:8776/v2/%\(tenant_id\)s \--region RegionOne volume
+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| adminurl     | http://controller:8776/v2/%(tenant_id)s |
| id           | 29b1496fed5145bfb6fafc90c265010f        |
| internalurl  | http://controller:8776/v2/%(tenant_id)s |
| publicurl    | http://controller:8776/v2/%(tenant_id)s |
| region       | RegionOne                               |
| service_id   | 546be85aaa664c71bc35add65c98a224        |
| service_name | cinder                                  |
| service_type | volume                                  |
+--------------+-----------------------------------------+

controller# openstack endpoint create \--publicurl http://controller:8776/v2/%\(tenant_id\)s \--internalurl http://controller:8776/v2/%\(tenant_id\)s \--adminurl http://controller:8776/v2/%\(tenant_id\)s \--region RegionOne volumev2
+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| adminurl     | http://controller:8776/v2/%(tenant_id)s |
| id           | 18777ee389dd4bfcbf5cc78920ce9f44        |
| internalurl  | http://controller:8776/v2/%(tenant_id)s |
| publicurl    | http://controller:8776/v2/%(tenant_id)s |
| region       | RegionOne                               |
| service_id   | 77ac9832fdcd485b928c118327027757        |
| service_name | cinderv2                                |
| service_type | volumev2                                |
+--------------+-----------------------------------------+
```

<!-- BREAK -->


### 14-3 パッケージのインストール

本書ではBlock StorageコントローラーとBlock Storageボリュームコンポーネントを一台のマシンで構築するため、両方の役割をインストールします。

```
controller# apt-get update
controller# apt-get install -y lvm2 cinder-api cinder-scheduler cinder-volume python-mysqldb python-cinderclient 
```


### 14-4 設定の変更

```
controller# vi /etc/cinder/cinder.conf

[DEFAULT]
...
verbose = True            ← 確認
auth_strategy = keystone  ← 確認
rpc_backend = rabbit      ← 追記

my_ip = 10.0.0.101   #controllerノード
enabled_backends = lvm
glance_host = controller

[oslo_messaging_rabbit]rabbit_host = controllerrabbit_userid = openstackrabbit_password = password

[oslo_concurrency]lock_path = /var/lock/cinder

[database]
connection = mysql://cinder:password@sql/cinder

[keystone_authtoken]auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = cinderpassword = password       ← cinderユーザーのパスワード(13-2で設定したもの)

[lvm]volume_driver = cinder.volume.drivers.lvm.LVMVolumeDrivervolume_group = cinder-volumesiscsi_protocol = iscsiiscsi_helper = tgtadm
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/cinder/cinder.conf | grep -v "^\s*$" | grep -v "^\s*#"
```


### 14-5 データベースに表を作成

```
controller# su -s /bin/sh -c "cinder-manage db sync" cinder
```

### 14-6 Cinderサービスの再起動

設定を反映させるために、Cinderのサービスを再起動します。

```
controller# service cinder-scheduler restart && service cinder-api restart
```

### 14-7 使用しないデータベースファイルの削除

```
controller# rm /var/lib/cinder/cinder.sqlite
```

<!-- BREAK -->

### 14-8 イメージ格納用ボリュームの作成

イメージ格納用ボリュームを設定するために物理ボリュームの設定、ボリューム作成を行います。

#### 14-8-1 物理ボリュームを追加

本例ではcontrollerノードにハードディスクを追加して、そのボリュームをCinder用ボリュームとして使います。controllerノードを一旦シャットダウンしてからハードディスクを増設し、再起動してください。新しい増設したディスクはdmesgコマンドやfdisk -lコマンドなどを使って確認できます。

仮想マシンにハードディスクを増設した場合は/dev/vdbなどのようにデバイス名が異なる場合があります。

#### 14-8-2 物理ボリュームを設定

以下コマンドで物理ボリュームを作成します。

+ LVM物理ボリュームの作成

```
controller# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
```

+ LVMボリュームグループの作成

```
controller# vgcreate cinder-volumes /dev/sdb
  Volume group "cinder-volumes" successfully created
```

#### 14-8-3 Cinder-Volumeサービスの再起動

Cinderストレージの設定を反映させるために、Cinder-Volumeのサービスを再起動します。

```
controller# service cinder-volume restart && service tgt restart
```

#### 14-8-4 admin環境変数設定ファイルの読み込み

Block StorageクライアントでAPI 2.0でアクセスするように環境変数設定ファイルを書き換えます。

```
controller# echo "export OS_VOLUME_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.sh
```

インスタンス格納用ボリュームを作成するために、admin環境変数を読み込みます。

```
controller# source admin-openrc.sh
```

<!-- BREAK -->

#### 14-8-5 ボリュームの作成

以下コマンドでインスタンス格納用ボリュームを作成します。


```
controller# cinder create --display-name testvolume01 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|              attachments              |                  []                  |
|           availability_zone           |                 nova                 |
|                bootable               |                false                 |
|          consistencygroup_id          |                 None                 |
|               created_at              |      2015-06-25T08:41:05.000000      |
|              description              |                 None                 |
|               encrypted               |                False                 |
|                   id                  | 776ed580-780e-431b-8d5a-f4d883b98884 |
|                metadata               |                  {}                  |
|              multiattach              |                False                 |
|                  name                 |             testvolume01             |
|         os-vol-host-attr:host         |                 None                 |
|     os-vol-mig-status-attr:migstat    |                 None                 |
|     os-vol-mig-status-attr:name_id    |                 None                 |
|      os-vol-tenant-attr:tenant_id     |   218010a87fe5477bba7f5e25c8211614   |
|   os-volume-replication:driver_data   |                 None                 |
| os-volume-replication:extended_status |                 None                 |
|           replication_status          |               disabled               |
|                  size                 |                  1                   |
|              snapshot_id              |                 None                 |
|              source_volid             |                 None                 |
|                 status                |               creating               |
|                user_id                |   9caffb5dc1d749c5b3e9493139fe8598   |
|              volume_type              |                 None                 |
+---------------------------------------+--------------------------------------+
```


#### 14-8-6 作成ボリュームの確認

以下コマンドで作成したボリュームを確認します。

```
controller# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 776ed580-780e-431b-8d5a-f4d883b98884 | available | testvolume01 |  1   |     None    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+

controller# cinder delete testvolume01
(作成したテストボリュームの削除)
```

※一覧にコマンドを実行して登録したボリュームが表示されて、ステータスがavailableとなっていれば問題ありません。

<!-- BREAK -->

## 15. Dashboardインストール・確認（controllerノード）

クライアントマシンからブラウザーでOpenStack環境を操作可能なWebインターフェイスをインストールします。

### 15-1 パッケージのインストール

controllerノードにDashboardをインストールします。

```
controller# apt-get update
controller# apt-get install -y openstack-dashboard
```

### 15-2 Dashboardの設定の変更

インストールしたDashboardの設定を変更します。

```
controller# vi /etc/openstack-dashboard/local_settings.py 

...
OPENSTACK_HOST = "controller"    ← 変更
ALLOWED_HOSTS = '*'              ← 確認

CACHES = {                       ← 確認'default': {'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache','LOCATION': '127.0.0.1:11211',   }}

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"  ← 変更
TIME_ZONE = "Asia/Tokyo"
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/openstack-dashboard/local_settings.py  | grep -v "^\s*$" | grep -v "^\s*#"
```

念のため、リダイレクトするように設定しておきます（数字は待ち時間）。

```
controller# vi /var/www/html/index.html
...
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="refresh" content="3; url=/horizon" />    ← 追記
```

変更した変更を反映させるため、Apacheとセッションストレージサービスを再起動します。

```
controller# service apache2 restart
```

<!-- BREAK -->


### 15-3 Dashboardへのアクセス確認

controllerノードとネットワーク的に接続されているマシンからブラウザで以下URLに接続してOpenStackのログイン画面が表示されるか確認します。

※ブラウザで接続するマシンは予めDNSもしくは/etc/hostsにcontrollerノードのIPを記述しておく等controllerノードの名前解決を行っておく必要があります。

```
http://controller/horizon/
```

※上記URLにアクセスしてログイン画面が表示され、ユーザーadminとdemoでログイン（パスワード:password）でログインできれば問題ありません。


<!-- BREAK -->

### 15-4 セキュリティグループの設定

OpenStackの上で動かすインスタンスのファイアウォール設定は、セキュリティグループで行います。ログイン後、次の手順でセキュリティグループを設定できます。

1.対象のユーザーでログイン<br>
2.「プロジェクト→コンピュート→アクセスとセキュリティ」を選択<br>
3.「ルールの管理」ボタンをクリック<br>
4.「ルールの追加」で許可するルールを定義<br>
5.「追加」ボタンをクリック<br>

セキュリティーグループは複数作成できます。作成したセキュリティーグループをインスタンスを起動する際に選択することで、セキュリティグループで定義したポートを解放したり、拒否したり、接続できるクライアントを制限することができます。

### 15-5 キーペアの作成

OpenStackではインスタンスへのアクセスはデフォルトで公開鍵認証方式で行います。次の手順でキーペアを作成できます。

1.対象のユーザーでログイン<br>
2.「プロジェクト→コンピュート→アクセスとセキュリティ」をクリック<br>
3.「キーペア」タブをクリック<br>
4.「キーペアの作成」ボタンをクリック<br>
5.キーペア名を入力<br>
6.「キーペアの作成」ボタンをクリック<br>
7.キーペア（拡張子:pem）ファイルをダウンロード<br>

インスタンスにSSH接続する際は、-iオプションでpemファイルを指定します。

```
client$ ssh -i mykey.pem cloud-user@instance-floating-ip  
```

<!-- BREAK -->

### 15-6 インスタンスの起動

前の手順でGlanceにCirrOSイメージを登録していますので、早速構築したOpenStack環境上でインスタンスを起動してみましょう。

1.対象のユーザーでログイン<br>
2.「プロジェクト→コンピュート→イメージ」をクリック<br>
3.イメージ一覧から起動するOSイメージを選び、「インスタンスの起動」ボタンをクリック<br>
4.「インスタンスの起動」詳細タブで起動するインスタンス名、フレーバー、インスタンス数を設定<br>
5.アクセスとセキュリティタブで割り当てるキーペア、セキュリティーグループを設定<br>
6.ネットワークタブで割り当てるネットワークを設定<br>
7.作成後タブで必要に応じてユーザーデータの入力（オプション）<br>
8.高度な設定タブでパーティションなどの構成を設定（オプション）<br>
9.右下の「起動」ボタンをクリック<br>

### 15-7 Floating IPの設定

起動したインスタンスにFloating IPアドレスを設定することで、Dashboardのコンソール以外からインスタンスにアクセスできるようになります。インスタンスにFloating IPを割り当てるには次の手順で行います。

1.対象のユーザーでログイン<br>
2.「プロジェクト→コンピュート→インスタンス」をクリック<br>
3.インスタンスの一覧から割り当てるインスタンスをクリック<br>
4.アクションメニューから「Floating IPの割り当て」をクリック<br>
5.「Floating IP割り当ての管理」画面のIPアドレスで「+」ボタンをクリック<br>
6.右下の「IPの確保」ボタンをクリック<br>
7.割り当てるIPアドレスとインスタンスを選択して右下の「割り当て」ボタンをクリック<br>

<!-- BREAK -->

### 15-8 インスタンスへのアクセス

Floating IPを割り当てて、かつセキュリティグループの設定を適切に行っていれば、リモートアクセスできるようになります。セキュリティーグループでSSHを許可した場合、端末からSSH接続が可能になります（下記は実行例）。

```
client$ ssh -i mykey.pem cloud-user@instance-floating-ip  
```

その他、適切なポートを開放してインスタンスへのPingを許可したり、インスタンスでWebサーバーを起動して外部PCからアクセスしてみましょう。

#Part.2 監視環境 構築編
<br>
構築したOpenStack環境をZabbixとHatoholで監視しましょう。

ZabbixはZabbix SIA社が開発・提供・サポートする、オープンソースの監視ソリューションです。
HatoholはProject Hatoholが開発・提供する、システム監視やジョブ管理やインシデント管理、ログ管理など、様々な運用管理ツールのハブとなるツールです。HatoholはZabbixやNagios、OpenStack Ceilometerに対応しており、これらのツールから情報を収集して性能情報、障害情報、ログなどを一括管理することができます。Hatoholのエンタープライズサポートはミラクル・リナックス株式会社が提供しています。

本編ではOpenStack環境を監視するためにZabbixとHatoholを構築するまでの流れを説明します。

<!-- BREAK -->

## 16. Zabbixのインストール

ZabbixはZabbix SIA社が提供するパッケージを使う方法とCanonical Ubuntuが提供するパッケージを使う方法がありますが、今回は新たなリポジトリー追加が不要なUbuntuが提供する標準パッケージを使って、Zabbixが動作する環境を作っていきましょう。

なお、UbuntuのZabbix関連のパッケージはuniverseリポジトリーで管理されています。universeリポジトリーを参照するように/etc/apt/sources.listを設定する必要があります。
次のように実行して同じような結果が出力されれば、universeリポジトリーが参照できるように設定されていると判断できます。

```
# apt-cache policy zabbix-server-mysql
zabbix-server-mysql:
  Installed: (none)
  Candidate: 1:2.2.2+dfsg-1ubuntu1
  Version table:
     1:2.2.2+dfsg-1ubuntu1 0
        500 http://us.archive.ubuntu.com/ubuntu/ trusty/universe amd64 Packages
```

本例ではZabbixをUbuntu Server 14.04.2上にオールインワン構成でセットアップする手順を示します。

### 16-1 パッケージのインストール
次のコマンドを実行し、ZabbixおよびZabbixの稼働に必要となるパッケージ群をインストールします。

```
zabbix# apt-get install -y php5-mysql zabbix-agent zabbix-server-mysql \
 zabbix-java-gateway zabbix-frontend-php
```

インストール中にMySQLのパスワードを設定する必要があります。

### 16-2 Zabbix用データベースの作成

#### 16-2-1 データベースの作成

次のコマンドを実行し、Zabbix用MySQLユーザおよびデータベースを作成します。

```
zabbix# mysql -u root -p << EOF
CREATE DATABASE zabbix CHARACTER SET UTF8;
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost' \
  IDENTIFIED BY 'zabbix';
EOF
Enter password: ← MySQLのrootパスワードを入力(16-1で設定したもの)
```

次のコマンドを実行し、Zabbix用データベースにテーブル等のデータベースオブジェクトを作成します。

```
zabbix# cd /usr/share/zabbix-server-mysql/
zabbix# zcat schema.sql.gz | mysql zabbix -uzabbix -pzabbix
zabbix# zcat images.sql.gz | mysql zabbix -uzabbix -pzabbix
zabbix# zcat data.sql.gz | mysql zabbix -uzabbix -pzabbix
```

#### 16-2-2 データベースの確認

作成したデータベーステーブルにアクセスしてみましょう。zabbixデータベースに様々なテーブルがあり、参照できれば問題ありません。

```
zabbix# mysql -u root -p
Enter password:         ← パスワードzabbixを入力
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| zabbix             |
+--------------------+
2 rows in set (0.00 sec)
mysql> use zabbix;
mysql> show tables;
+-----------------------+
| Tables_in_zabbix      |
+-----------------------+
| acknowledges          |
| actions               |
| alerts                |
...

mysql> describe acknowledges;
+---------------+---------------------+------+-----+---------+-------+
| Field         | Type                | Null | Key | Default | Extra |
+---------------+---------------------+------+-----+---------+-------+
| acknowledgeid | bigint(20) unsigned | NO   | PRI | NULL    |       |
| userid        | bigint(20) unsigned | NO   | MUL | NULL    |       |
| eventid       | bigint(20) unsigned | NO   | MUL | NULL    |       |
| clock         | int(11)             | NO   | MUL | 0       |       |
| message       | varchar(255)        | NO   |     |         |       |
+---------------+---------------------+------+-----+---------+-------+
5 rows in set (0.01 sec)
```

<!-- BREAK -->

### 16-3 Zabbixサーバーの設定および起動
/etc/zabbix/zabbix_server.confを編集し、次の行を追加します。なお、MySQLユーザzabbixのパスワードを別の文字列に変更した場合は、該当文字列を指定する必要があります。

```
zabbix# vi /etc/zabbix/zabbix_server.conf
...
DBPassword=zabbix
```

/etc/default/zabbix-serverを編集し、起動可能にします。

```
zabbix# vi /etc/default/zabbix-server
...
# Instructions on how to set up the database can be found in
# /usr/share/doc/zabbix-server-mysql/README.Debian
START=yes                     ← noからyesに変更
```

以上の操作を行ったのち、サービスzabbix-serverを起動します。

```
zabbix# service zabbix-server restart
```

<!-- BREAK -->

### 16-4 Zabbix frontendの設定および起動
PHPの設定をZabbixが動作するように修正するため、/etc/php5/apache2/php.iniを編集します。
 
 ```
 zabbix# vi /etc/php5/apache2/php.ini

 [PHP]
 ...
 post_max_size = 16M          ← 変更
 max_execution_time = 300     ← 変更
 max_input_time = 300         ← 変更
 
 [Date]
 date.timezone = Asia/Tokyo   ← 変更
 ```

Zabbix frontendへアクセスできるよう、設定ファイルをコピーします。

```
zabbix# cp -p /usr/share/doc/zabbix-frontend-php/examples/apache.conf /etc/apache2/conf-enabled/zabbix.conf
```

これまでの設定変更を反映させるため、サービスApache2をリロードします。

```
zabbix# service apache2 reload
```

次に、Zabbix frontendの接続設定を行います。次のコマンドを実行し、一時的に権限を変更します。

```
zabbix# chmod 775 /etc/zabbix
zabbix# chgrp www-data /etc/zabbix
```

WebブラウザでZabbix frontendへアクセスします。画面指示に従い、Zabbixの初期設定を行います。

```
http://<Zabbix frontendのIPアドレス>/zabbix/
```

次のような画面が表示されます。「Next」ボタンをクリックして次に進みます。

![Zabbix初期セットアップ](./images/zabbix-setup.png)

- 「2. Check of pre-requisites」は、システム要件を満たしている（全てOKとなっている）ことを確認します。
- 「3. Configure DB connection」は次のように入力し、「Test connection」ボタンを押してOKとなることを確認します。

項目          | 設定値
------------- | -------------------------------
Database type | MySQL
Database host | localhost
Database Port | 0
Database name | zabbix
User          | zabbix
Password      | zabbix

- 「4. Zabbix server details」はZabbix Serverのインストール場所の指定です。本例ではそのまま次に進みます。
- 「5. Pre-Installation summary」で設定を確認し、問題なければ次に進みます。
- 「6. Install」で設定ファイルのパスが表示されるので確認し「Finish」ボタンをクリックします（/etc/zabbix/zabbix.conf.php）。
- ログイン画面が表示されるので、Admin/zabbix（初期パスワード）でログインします。

Zabbixの初期セットアップ終了後にログイン画面が表示されますので、実際に運用開始する前に次のコマンドを実行して権限を元に戻します。

```
zabbix# chmod 755 /etc/zabbix
zabbix# chgrp root /etc/zabbix
```

<!-- BREAK -->

## 17. Hatoholのインストール

HatoholはCentOS6.5以降、Ubuntu Server 12.04および14.04などで動作します。
CentOS6.5以降および7.x向けには導入に便利なRPMパッケージが公式で提供されています。

本例ではHatoholをCentOS 7上にオールインワン構成でセットアップする手順を示します。

![Hatoholダッシュボード](./images/hatohol2.png)

### 17-1 インストール

　1. Hatoholをインストールするために、Project Hatohol公式のYUMリポジトリーを登録します。

```
hatohol# wget -P /etc/yum.repos.d/ http://project-hatohol.github.io/repo/hatohol-el7.repo
```

　2. EPELリポジトリー上のパッケージのインストールをサポートするため、EPELパッケージを追加インストールします。

```
hatohol# yum install -y epel-release
hatohol# yum update
```

　3. Hatoholサーバをインストールします。

```
hatohol# yum -y install hatohol-server
```

　4. Hatohol Web Frontendをインストールします。

```
hatohol# yum install -y hatohol-web
```

　5. 必要となる追加パッケージをインストールします。

```
hatohol# yum install -y mariadb-server qpid-cpp-server
```


### 17-2 セットアップ

　1. /etc/my.cnfの編集

 * 編集前

```
!includedir /etc/my.cnf.d
```

 * 編集後

```
includedir /etc/my.cnf.d
```

<!-- BREAK -->

　2. /etc/my.cnf.d/server.cnfの編集  

セクション[mysqld]に、少なくとも次のパラメータを追記します。

```
[mysqld]
character-set-server = utf8
skip-character-set-client-handshake
default-storage-engine = innodb
innodb_file_per_table
```

　3. MariaDBサービスの自動起動の有効化と起動

```
hatohol# systemctl enable mariadb.service
hatohol# systemctl start mariadb.service
```

　4. MariaDBユーザrootのパスワード変更

```
hatohol# mysqladmin password
```

　5. Hatohol DBの初期化

```
hatohol# hatohol-db-initiator --db_user root --db_password <4で設定したrootパスワード>
...
Succeessfully loaded: /usr/bin/../share/hatohol/sql/init-user.sql
Succeessfully loaded: /usr/bin/../share/hatohol/sql/server-type-zabbix.sql
Succeessfully loaded: /usr/bin/../share/hatohol/sql/server-type-nagios.sql
Succeessfully loaded: /usr/bin/../share/hatohol/sql/server-type-hapi-zabbix.sql
Succeessfully loaded: /usr/bin/../share/hatohol/sql/server-type-hapi-json.sql
Succeessfully loaded: /usr/bin/../share/hatohol/sql/server-type-ceilometer.sql
```

初期状態で上記コマンドを実行した場合、MySQLユーザhatohol、データベースhatoholが作成されます。これらを変更する場合、事前に/etc/hatohol/hatohol.confを編集してください。

　6. Hatohol Web用DBの作成

```
hatohol# mysql -u root -p << EOF
CREATE DATABASE hatohol_client;
GRANT ALL PRIVILEGES ON hatohol_client.* TO 'hatohol'@'localhost' \
  IDENTIFIED BY 'hatohol';
EOF
```

　7. Hatohol Web用DBへのテーブル追加

```
hatohol# /usr/libexec/hatohol/client/manage.py syncdb
```

　8. Hatoholサーバーの自動起動の有効化と起動

```
hatohol# systemctl enable hatohol.service
hatohol# systemctl start hatohol.service
```

　9. Hatohol Webの自動起動の有効化と起動

```
hatohol# systemctl enable httpd.service
hatohol# systemctl start httpd.service
```

　10. HatoholおよびApache Webサーバーの動作確認

```
hatohol# systemctl status -l hatohol.service
hatohol# systemctl status -l httpd.service
```

<!-- BREAK -->

### 17-3 セキュリティ設定の変更

CentOSインストール後の初期状態では、SElinux, Firewalld, iptablesといったセキュリティ機構により他のコンピュータからのアクセスに制限が加えられます。Hatoholを使用するにあたり、これらを適切に解除する必要があります。

　1. SELinuxの設定  

```
hatohol# getenforce
Enforcing
```

Enforcingの場合、次のコマンドでSElinuxポリシールールの強制適用を解除できます。

```
hatohol# setenforce 0
hatohol# getenforce
Permissive
```

恒久的にSELinuxポリシールールの適用を無効化するには、/etc/selinux/configを編集します。

 * 編集前

 ```
 SELINUX=enforcing
 ```

 * 編集後

 ```
 SELINUX=permissive
 ```
 
 完全にSELinuxを無効化するには、次のように設定します。

 ```
 SELINUX=disabled
 ```
 
 
 ```
 筆者注:
 SELinuxはできる限り無効化すべきではありません。
 ```

　2. パケットフィルタリングの設定
フィルタリングの設定変更は、次のコマンドで恒久的に変更可能です。

```
hatohol# firewall-cmd --add-service=http --zone=public
hatohol# iptables-save > /etc/sysconfig/iptables
```

<!-- BREAK -->

### 17-4 Hatoholによる情報の閲覧

Hatohol Webが動作しているホストのトップディレクトリーをWebブラウザで表示してください。10.0.0.10で動作している場合は、次のURLとなります。admin/hatohol（初期パスワード）でログインできます。

```
http://10.0.0.10/
```

Hatoholは監視サーバーから取得したログ、イベント、性能情報を表示するだけでなく、それらの情報を統計してグラフとして出力することができる機能が備わっています。CPUのシステム時間、ユーザー時間をグラフとして出力すると次のようになります。

![Hatoholのグラフ機能](./images/hatohol3.png)

<!-- BREAK -->

### 17-5 HatoholにZabbixサーバーを登録

Hatoholをインストールできたら、Zabbixサーバーの情報を追加します。Hatohol Webにログインしたら、上部のメニューバーの「設定→監視サーバー」をクリックします。「監視サーバー」の画面に切り替わったら「監視サーバー追加」ボタンをクリックしてノードを登録します。

項目               | 設定値
------------------ | -------------------------------
監視サーバータイプ | Zabbix
ニックネーム       | zabbix1
ホスト名           | zabbix
IPアドレス         | (ZabbixサーバーのIPアドレス)
ポート番号         | 80
ユーザー           | Admin
パスワード         | zabbix

ページを再読み込みして、通信状態が「初期状態」から「正常」になる事を確認します。

![Zabbixサーバーの追加](./images/hatohol1.png)

<!-- BREAK -->

### 17-6 HatoholでZabbixサーバーの監視

インストール直後のZabbixサーバーはモニタリング設定が無効化されています。これを有効化するとZabbixサーバー自身の監視データを取得する事ができるようになり、Hatoholで閲覧できるようになります。

Zabbixサーバーのモニタリング設定を変更するには、次の手順で行います。

+ Zabbixのメインメニュー「Configuration → Host groups」をクリックします。
+ Host groups一覧から「Zabbix server」をクリックします。
+ 「Zabbix server」のHostの設定で、Statusを「Monitored」に変更します。
+ 「Save」ボタンをクリックして設定変更を適用します。

以上の手順で、Zabbixサーバーを監視対象として設定できます。

### 17-7 Hatoholでその他のホストの監視

ZabbixとHatoholの連携ができたので、あとは対象のサーバーにZabbix Agentをインストールし、手動でZabbixサーバーにホストを追加するか、ディスカバリ自動登録を使って、特定のネットワークセグメントに所属するZabbix Agentがインストールされたホストを自動登録するようにセットアップするなどの方法で監視ノードを追加できます。
追加したノードはZabbixおよびHatoholで監視する事ができます。

#### 17-7-1 Zabbix Agentのインストール

ZabbixでOpenStackのcontrollerノード、networkノード、computeノードを監視するためにZabbix Agentをインストールします。Ubuntuには標準でZabbix Agentパッケージが用意されているので、apt-getコマンドなどを使ってインストールします。

```
# apt-get update && apt-get install -y zabbix-agent
```

#### 17-7-2 Zabbix Agentの設定

Zabbix Agentをインストールしたら次にどのZabbixサーバーと通信するのか設定を行う必要があります。最低限必要な設定は次の3つです。次のように設定します。

(controllerノードの設定記述例)

```
# vi /etc/zabbix/zabbix_agentd.conf
...
Server          10.0.0.10     ← ZabbixサーバーのIPアドレスに書き換え
ServerActive    10.0.0.10     ← ZabbixサーバーのIPアドレスに書き換え
Hostname  controller      ← Zabbixサーバーに登録する際のホスト名と同一のものを設定
ListenIP  10.0.0.101      ← Zabbixエージェントが待ち受ける側のIPアドレス
```

ListenIPに指定するのはZabbixサーバーと通信できるNICに設定したIPアドレスを設定します。

<!-- BREAK -->

変更したZabbix Agentの設定を反映させるため、Zabbix Agentサービスを再起動します。

```
# service zabbix-agent restart
```

#### 17-7-3 ホストの登録

Zabbix Agentのセットアップが終わったら、次にZabbix AgentをセットアップしたサーバーをZabbixの管理対象として追加します。次のように設定します。

- 「Configuration → Host」をクリックします。初期設定時はZabbix serverのみが登録されていると思います。同じように監視対象サーバーをZabbixに登録します。

- 「Hosts」画面の右上にある、「Create Host」ボタンをクリックします。
- 次のように設定します。

「Host」の設定    | 説明
----------------- | ----------------
Host name         | zabbix_agentd.confにそれぞれ記述したHostnameを記述
Visible name      | 表示名（オプション）
Groups            | 所属グループの指定。例としてLinux serversを指定
Agent interfaces  | 監視対象とするAgentがインストールされたホストのIPアドレス（もしくはホスト名）
Status            | Monitored

その他の項目は適宜設定します。

- 「CONFIGURATION OF HOSTS」の「Templates」タブをクリックして設定を切り替えます。
- 「Link new templates」の検索ボックスに「Template OS Linux」と入力し、選択肢が出てきたらクリックします。そのほかのテンプレートを割り当てるにはテンプレートを検索し、該当のものを選択します。
- 「Link new templates」にテンプレートを追加したら、その項目の「Add」リンクをクリックします。「Linked templates」に追加されます。
- 「Save」ボタンをクリックします。
- 「Hosts」画面にサーバーが追加されます。ページの再読み込みを実行して、Zabbixエージェントが有効になっていることを確認してください。「Z」アイコンが緑色になればOKです。

![Zabbixエージェントステータスを確認](./images/zabbix-agent.png)

- ほかに追加したいサーバーがあれば「Zabbix Agentのインストール、設定、ホストの登録」の一連の流れを繰り返します。監視したい対象が大量にある場合はオートディスカバリを検討してください。

<!-- BREAK -->

#### 17-7-4 Hatoholで確認

登録したサーバーの情報がHatoholで閲覧できるか確認してみましょう。Zabbixサーバー以外のログなど表示できるようになればOKです。

![OpenStackノードの監視](./images/hatohol-view.png)


#### 17-7-5 参考情報

ホストの追加やディスカバリ自動登録については次のドキュメントをご覧ください。

- <https://www.zabbix.com/documentation/2.2/jp/manual/quickstart/host>
- <https://www.zabbix.com/documentation/2.2/jp/manual/discovery/auto_registration>
- <http://www.zabbix.com/jp/auto_discovery.php>
- <https://www.zabbix.com/documentation/2.2/jp/manual/discovery/network_discovery/rule>

<!-- BREAK -->

### 17-8 Hatohol Arm Plugin Interfaceを使用する場合の操作
Hatohol Arm Plugin Interface(HAPI)を使用する場合、/etc/qpid/qpidd.confに次の行を追記します。なお、=の前後にスペースを入れてはなりません。

```
auth=no
```