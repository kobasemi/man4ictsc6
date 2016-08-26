# Squidサーバの構築

OS: CentOS7

まず，以下のコマンドでSquidと関連パッケージをインストールする．

```
# yum install squid
```

Squidの設定ファイルは`/etc/squid/`以下にあり，メインの設定ファイルは`squid.conf`である．これを以下のように編集する．

```
#
# Recommended minimum configuration:
#

# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 10.0.0.0/8     # RFC1918 possible internal network
acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http
acl CONNECT method CONNECT

#
# Recommended minimum Access Permission configuration:
#
# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager

# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
http_access deny to_localhost

#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#

# URL filtering sample
# dstdomain: destination domain
# Include subdomain by beginning dot
acl blacklist dstdomain .example.com example.net
http_access deny blacklist

# Example rule allowing access from your local networks.
# Adapt localnet in the ACL section to list your (internal) IP networks
# from where browsing should be allowed
http_access allow localnet
http_access allow localhost

# And finally deny all other access to this proxy
http_access deny all

# Squid normally listens to port 3128
http_port 3128

# Memory size for caching
cache_mem 16 MB

# Set maximum cache size per object
maximum_object_size 16384 KB

# Uncomment and adjust the following to add a disk cache directory.
#
# 1000: disk cache size (MB)
# 16: the number of primary directory
# 256: the number of secondary directory
cache_dir ufs /var/spool/squid 1000 16 256

# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid

#
# Add any of your own refresh_pattern entries above these.
#
# Configure refresh pattern of cache
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern ^gopher:        1440    0%      1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
refresh_pattern -i \.(gif|png|jpg|jpeg|ico)$ 10080 90% 43200 override-expire ignore-no-store ignore-private
refresh_pattern -i \.(iso|avi|wav|mp3|mp4|mpeg|swf|flv|x-flv)$ 43200 90% 432000 override-expire ignore-no-store ignore-private
refresh_pattern -i \.(deb|rpm|exe|zip|tar|tgz|ram|rar|bin|ppt|doc|tiff)$ 10080 90% 43200 override-expire ignore-no-store ignore-private
refresh_pattern -i \.index.(html|htm)$ 0 40% 10080
refresh_pattern -i \.(html|htm|css|js)$ 1440 40% 40320


# To anonymous
visible_hostname unknown
forwarded_for off
request_header_access From deny all
request_header_access Referer deny all
request_header_access User-Agent deny all
request_header_access X-FORWARDED-FOR deny all
request_header_access Via deny all
```

`refresh_pattern`ではキャッシュの更新ルールを設定する．

- `-i`オプションを指定することで，後ろに指定するキャッシュ対象となるファイルをフィルタする正規表現についてcase insensitiveであると設定できる．

- その次にはそのキャッシュが新しいものであるかの期限を分単位で指定する．例えば上の`.jpg`を例にとると，対象となるオブジェクトに対応するキャッシュの保存期間が10080分未満であればまだ新しいとみなしてキャッシュをそのまま渡す．

- その次のパーセンテージで指定する部分だが，キャッシュの保存期間が`左の数値 <= 保存期間 <= 右の数値`である場合に，キャッシュサーバ上でのオブジェクトの保存時間をオリジナルのオブジェクトの作成・変更からの経過時間で割った値がここで指定したパーセント値より小さい場合にキャッシュがまだ新しいとみなしてキャッシュをそのまま渡す．それ以外はオリジナルのオブジェクトのコピーを取得して更新する．

- 最後の数値はキャッシュが古いものかどうかの期限を分単位で指定する．例えば上の`.jpg`を例にとると，対象となるオブジェクトに対応するキャッシュの保存期間が43200分より長い場合にキャッシュが古くなったと判定し，オリジナルのオブジェクトのコピーを取得して更新する．


設定ファイルを作成できたら，キャッシュディレクトリを以下のコマンドで作成する．

```
# squid -zN
```

また，以下のコマンドで設定ファイルのエラーチェックをする．

```
# squid -k parse
```

設定ファイルの書き方で不味い部分があると，`WARNING`などで警告を表示してくれる．内容を確認して適宜修正する．

次に，firewalldでsquidへの接続を許可するルールを追加する．まずは`/etc/firewalld/services/squid.xml`を以下の内容で作成する．

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>squid</short>
  <description>Caching proxy for the Web</description>
  <port protocol="tcp" port="3128"/>
</service>
```

作成できたら，以下のコマンドで作成した定義ファイルを読み込ませ，publicゾーンにSquidのルールを追加する．

```
# firewall-cmd --reload
# firewall-cmd --zone=public --add-service=squid
# firewall-cmd --zone=public --add-service=squid --permanent
```

firewalldにルールを追加できたら，以下のコマンドでSquidを起動し，システム起動時にも自動起動するように設定する．

```
# systemctl start squid
# systemctl enable squid
```

## 参考

- [squid : refresh_pattern configuration directive](http://www.squid-cache.org/Doc/config/refresh_pattern/)

- [squid : request_header_access configuration directive](http://www.squid-cache.org/Doc/config/request_header_access/)

- [Leverage OSS：Squidの更新パターンでインターネットアクセスを高速化する (1/2) - ITmedia エンタープライズ](http://www.itmedia.co.jp/enterprise/articles/0812/01/news024.html)

  ​