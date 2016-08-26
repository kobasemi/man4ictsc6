# firewalld

CentOS 7から使われるようになった．`iptables`のラッパーで，`iptables`より直感的かつシンプルにファイアウォールを設定出来る．

ネットワークコネクションやインターフェイスの信頼度を定義するゾーンという考え方が取り入られている．ユーザは，そのなかから自分に合ったゾーンを自由に選択し，必要であれば設定を編集することも出来る．



## ゾーン一覧

### drop (変更不可)

内向きの通信を破棄する．ただし外向きに出て行った通信の戻りパケットは許可する．

### block (変更不可)

全てのコネクションを破棄する．

### public

デフォルトで選択されているゾーン．`ssh`と`dhcpv6-client`のみが許可されている．

### external

IPマスカレードが有効になっている．また`ssh`のみ許可されている．

### dmz

`ssh`のみが許可されている．

### work

`ssh`と`ipp-client`，`dhcpv6-client`が許可されている．

### home

`ssh`と`mdns`，`samba-client`，`ipp-client`，`dhcpv6-client`が許可されている．

### internal

`ssh`と`mdns`，`samba-client`，`ipp-client`，`dhcpv6-client`が許可されている．

### trusted (変更不可)

全てのコネクションを許可する．



## 基本的なコマンド（`firewall-cmd`）の使い方

### `--state`

`firewalld`の起動状態を確認

```shell
# firewall-cmd --state
running
```



### `--reload`

ファイアウォールの設定を反映

```shell
# firewall-cmd --reload
success
```



### `--get-zones`

サポートされている全てのゾーンを一覧

```shell
# firewall-cmd --get-zones
block dmz drop external home internal public trusted work
```



### `--get-services`

サポートされている全てのサービスを一覧

```shell
# firewall-cmd --get-services
RH-Satellite-6 amanda-client bacula bacula-client dhcp dhcpv6 dhcpv6-client dns freeipa-ldap freeipa-ldaps freeipa-replication ftp high-availability http https imaps ipp ipp-client ipsec iscsi-target kerberos kpasswd ldap ldaps libvirt libvirt-tls mdns mountd ms-wbt mysql nfs ntp openvpn pmcd pmproxy pmwebapi pmwebapis pop3s postgresql proxy-dhcp radius rpc-bind rsyncd samba samba-client smtp ssh telnet tftp tftp-client transmission-client vdsm vnc-server wbem-https
```



### `--get-icmptypes`

サポートされている全てのICMPタイプを一覧

```shell
# firewall-cmd --get-icmptypes
destination-unreachable echo-reply echo-request parameter-problem redirect router-advertisement router-solicitation source-quench time-exceeded
```



### `--list-all-zones`

全てのゾーンと各ゾーンで有効な機能を一覧

```shell
# firewall-cmd --list-all-zones
...
```



### `[--zone=<zone>] --list-all`

その`<zone>`において有効な機能を表示（省略された場合はデフォルトゾーン）

```shell
# firewall-cmd --list-all
public (default, active)
  interfaces: ens32
  sources:
  services: dhcpv6-client ssh
  ports:
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```



### `[--zone=<zone>] --list-services`

その`<zone>`において有効なサービスを一覧

```shell
# firewall-cmd --list-services
dhcpv6-client ssh
```



### `--get-default-zone`

デフォルトのゾーンを取得

```shell
# firewall-cmd --get-default-zone
public
```



### `--get-active-zones`

アクティブ状態にあるゾーンを取得

```shell
# firewall-cmd --get-active-zones
public
  interfaces: ens32
```



### `--set-default-zone=<zone>`

デフォルトゾーンとして，`<zone>`を設定

```shell
# firewall-cmd --set-default-zone=public
success
```



### ` [--zone=<zone>] --add-interface=<interface>`

その`<zone>`に対して，インターフェイスがなければ`<interface>`を追加

```shell
# firewall-cmd --add-interface=ens32
success
```



### ` [--zone=<zone>] --change-interface=<interface>`

その`<zone>`に対して，インターフェイスが存在すれば`<interface>`に変更

```shell
# firewall-cmd --add-interface=ens32
success
```



### ` [--zone=<zone>] --add-interface=<interface>`

その`<zone>`に対して，関連付けられた`<interface>`を削除

```shell
# firewall-cmd --add-interface=ens32
success
```



## ファイアウォールのルールの操作

ルールを追加するとき，そのルールが永続的に設定するものか，一時的に設定するものかを選択することが出来る．一時的に設定したものは，リロード（`--reload`）や再起動後に変更が失われるが，永続的に設定したものは，変更が失われない．

ルールを永続的に設定する場合は，コマンドのオプションに`--permanent`を付与する．これで，そのルールは永続化され，リロードや再起動があっても維持されたままになる．永続化する場合は，タイムアウトのオプション（後述）を付与することは出来ない．



### `[--zone=<zone>] --add-service=<service> [--timeout=<seconds>]`

その`<zone>`に対して，指定した`<service>`を有効化（タイムアウトが省略された場合は，変更が失われるまで有効）

```shell
# # HTTPサービスを有効化
# firewall-cmd --add-service=http
success
# # HTTPサービスを60秒だけ有効化
# firewall-cmd --add-service=http --timeout=60
success
```



### `[--zone=<zone>] --remove-service=<service>`

その`<zone>`に対して，指定した`<service>`を無効化

```shell
# firewall-cmd --remove-service=http
success
```



### `[--zone=<zone>] --add-port=<port>[-<port>]/<protocol> [--timeout=<seconds>]`

その`<zone>`に対して，指定した`<port>`（`-<port>`とすることで範囲指定可能）かつ`<protocol>`を有効化（プロトコルは，`tcp`か`udp`）

```shell
# # TCPの80番ポートを有効化
# firewall-cmd --add-port=80/tcp
success
# # TCPの60000から60010番ポートまで有効化
# firewall-cmd --add-port=60000-60010/tcp
success
```



### `[--zone=<zone>] --remove-port=<port>[-<port>]/<protocol>`

その`<zone>`に対して，指定した`<port>`と`<protocol>`の組み合わせを無効化

```shell
# firewall-cmd --remove-port=80/tcp
success
# firewall-cmd --remove-port=60000-60010/tcp
success
```



### `[--zone=<zone>] --add-icmp-block=<icmptype>`

その`<zone>`に対して，指定した`<imcptype>`の通信をブロック

```shell
# # ICMPパケットのecho-replayに関するものをブロック
# firewall-cmd --add-icmp-block=echo-reply
success
```



### `[--zone=<zone>] --remove-icmp-block=<icmptype>`

その`<zone>`に対して，指定した`<imcptype>`の通信をブロックする設定を削除

```shell
# firewall-cmd --remove-icmp-block=echo-reply
success
```



## 設定ファイル

`/etc/firewalld/`以下にある．



## 参考サイト

- [https://fedoraproject.org/wiki/FirewallD/jp](https://fedoraproject.org/wiki/FirewallD/jp)
- [http://www.unix-power.net/centos7/firewalld.html](http://www.unix-power.net/centos7/firewalld.html)

