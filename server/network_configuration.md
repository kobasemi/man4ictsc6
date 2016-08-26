# Network Configuration

CentOS 6と7，Ubuntu 14.04.4と16.04のそれぞれについて，ネットワークの設定方法をまとめる．

## CentOS6

`/etc/sysconfig/network-scripts/ifcfg-eth1`を次のように設定する．

### DHCPを使う場合

```
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=dhcp
```

### 静的IPアドレスを割り当てる場合

```
NAME=eth1
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.100.1
NETMASK=255.255.255.0
BROADCAST=192.168.100.255
GATEWAY=192.168.100.254
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
IPV6_AUTOCONF=no
IPV6_DEFROUTE=no
IPV6_FAILURE_FATAL=no
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
PEERDNS=yes
PEERROUTES=yes
IPV6_PRIVACY=no
DNS1=192.168.2.80
DNS2=192.168.2.81
```

### 設定の適用

以下のコマンドで，ネットワーク設定を適用する．

```
# service network restart
```

## CentOS7

`/etc/sysconfig/network-scripts/ifcfg-ens32`を設定する．

*設定方法はCentOS6と同じなので，割愛*

### 設定の適用

以下のコマンドで，ネットワーク設定を適用する．

```
# systemctl restart network
```

### nmtui

```
# nmtui
```

これで，Text User Interfaceからグラフィカルにネットワーク設定が可能．

### nmcli

先述したネットワーク設定ファイルをコマンド越しに変更できるもの．直接設定ファイルを編集した方が早い．

## Ubuntu14.04.4

`/etc/network/interfaces`を編集する．

### DHCPを使う場合

```
auto eth0
iface eth0 inet dhcp
```

### 静的IPアドレスを割り当てる場合

```
auto eth0
iface eth0 inet static
address 192.168.2.80
netmask 255.255.255.0
broadcast 192.168.2.255
gateway 192.168.2.254
dns-nameservers 192.168.2.80 192.168.2.81
```

### 設定の適用

```
# ifdown eth0 && ifup eth0
```

## Ubuntu16.04

`/etc/network/interfaces`を編集する．

*設定方法はUbuntu14.04.4と同じなので，割愛する．*

### 設定の適用

```
# systemctl restart networking
```

その後，以下のコマンドでIPアドレスを確認する．

```
$ ip a
```

DHCPで割り当てられたアドレスが残っていた場合，以下のコマンドでシステムを再起動する．

```
# shutdown -r now
```

