# DHCPサーバ構築

OS：CentOS6

## ソフトウェアのインストール

以下のコマンドを実行して，必要なソフトウェアをインストールする．

```
# yum install dhcp
```

## DHCPサーバ設定

DHCPサーバの設定ファイルは，`/etc/dhcp/`以下にある．このうちIPv4のDHCPサーバの設定ファイルは`/etc/dhcp/dhcpd.conf`なので，これを以下のように編集する．

```
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.sample
#   see 'man 5 dhcpd.conf'
#
# 提供するDNSサーバアドレスを指定する
option domain-name-servers 192.168.2.80, 192.168.2.81;

# クライアントのサブネットマスクを指定する
option subnet-mask 255.255.255.0;

# デフォルトの貸し時間を指定
default-lease-time 600;

# クライアントが期限を求めた場合の貸し時間を指定
max-lease-time 7200;

# このDHCPサーバが正当で，信頼できる権威のあるものだと宣言する
# クライアントが要求したIPアドレスが正当でないとサーバが判断するとDHCPNAKを返す
authoritative;

# syslog設定
log-facility local0;

subnet 192.168.2.0 netmask 255.255.255.0 {
  range 192.168.2.100 192.168.2.200;
  option routers 192.168.2.254;
}
```

また，DHCPサーバを提供するネットワークインタフェースを指定するために，`/etc/sysconfig/dhcpd`に以下のものを追記する．

```
DHCPDARGS=eth1
```

## syslog設定

`/etc/dhcp/dhcpd.conf`にて`log-facility local0;`と設定したので，`/etc/rsyslog.conf`に次の内容を追記する．

```
local0.*		/var/log/dhcpd.log
```

また，DHCPD関係のログが`/var/log/messages`に出力されないように，`/var/log/messages`の箇所を次のようにする．

```
*.info;mail.none;authpriv.none;cron.none;local0.none                /var/log/messages
```

`/var/log/dhcpd.log`がログローテートの対象となるように，`/etc/logrotate.d/syslog`に以下のものを追記する．

```
/var/log/dhcpd.log
```

編集が終わったら，rsyslogをリスタートし，DHCPサービスを起動する．

```
# service rsyslog restart
# service dhcpd start
```

起動が確認できたら，システム起動時に自動的にサービスが起動するように設定する．

```
# chkconfig dhcpd on
```

## 参考

- [dhcpd - ArchWiki](https://wiki.archlinuxjp.org/index.php/Dhcpd)
- [DHCPサーバー構築(dhcp) - CentOSで自宅サーバー構築](https://centossrv.com/dhcp.shtml)

