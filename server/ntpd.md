# NTPサーバの構築

OS: CentOS7

## 必要なソフトウェアのインストール

以下のコマンドを実行し，NTPサーバの構築に必要なソフトウェアをインストールする．

```
# yum install ntp
```

## NTPサーバの設定

NTPサーバの設定ファイルは`/etc/ntp.conf`である．これを以下のように編集する．

```
server 133.243.238.163 iburst minpoll 4 maxpoll 17 prefer
server 133.243.238.164 iburst minpoll 4 maxpoll 17
server 133.243.238.243 iburst minpoll 4 maxpoll 17
server 133.243.238.244 iburst minpoll 4 maxpoll 17

driftfile /var/lib/ntp/drift
logfile /var/log/ntp.log

# Blocking Unauthorized Access
restrict default ignore

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1
restrict ::1

restrict 133.243.238.163 nomodify nopeer noquery notrap
restrict 133.243.238.164 nomodify nopeer noquery notrap
restrict 133.243.238.243 nomodify nopeer noquery notrap
restrict 133.243.238.244 nomodify nopeer noquery notrap

restrict 192.168.2.0 mask 255.255.255.0 limited kod nomodify notrap nopeer noquery

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).

#broadcast 192.168.1.255 autokey        # broadcast server
#broadcastclient                        # broadcast client
#broadcast 224.0.1.1 autokey            # multicast server
#multicastclient 224.0.1.1              # multicast client
#manycastserver 239.255.254.254         # manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography.
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor

# Add local clock, in case this server loses internet access, it will continue serving time to the network
# add local clock as a stratum 10 server (using the fudge command) so that it will never be used unless internet access is lost
server 127.127.1.1
fudge 127.127.1.1 stratum 12
```

### restrictのオプション

- `ignore`：全てのNTPパケットに応答しない．ntpqやntpdcのクエリも含む
- `kod`：limitオプションと共に使用する．レートリミットを超えたクエリは破棄されるが，それに加えてkiss-o'-death（KoD）パケットがクライアントに送信される．これはクライアントに対して「もう問い合わせを止めろ」という意向を伝えるもので，クライアントは断られたことを示すフラグを立ててそのサーバへの問い合わせをしない決まりになっている
- `limited`：レートリミットを超えた頻度の問い合わせに対し，参照サービスの提供を拒絶する．クライアントからの問い合わせの履歴を保持する必要があることから，これを指定したrestrictセクションが一つでもあるとdisable monitorは無効となる
- `nomodify`：設定や状態が変更されるようなntpqとntpdcのクエリが来た場合に拒否する
- `noquery`：ntpqとntpdcクエリ，つまりNTPサーバの設定変更や状態の確認・変更のパケットを受け付けない
- `nopeer`：NTPサーバをグループ化するような（Symmetric Activeのような）パケットを拒否する
- `noserve`：ntpqとntpdcクエリのみ受け付け，他のパケットを全て拒否する（!noquery）
- `notrap`：ホストに対してNTP内部情報を提供しない
- `notrust`：Ver4.1以前は時間の参照源として信用せず使わないがその他の制限は無し，Ver4.2以降は参照源としてだけでなくクライアントとしても暗号認証による信頼関係がない限り信用しない
- `version`：サーバのNTPバージョンと一致しないクエリを拒否する

## NTPサーバ起動

起動する前に，まず手動で時刻を合わせる．

```
# ntpdate ntp.nict.jp
```

そして，以下のコマンドでNTPサーバを起動する．

```
# systemctl start ntpd
```

起動後，以下のコマンドでサーバの状態を確認する．

```
# ntpq -p
```

サーバを起動してすぐは，以下のような状態．

```
[root@ntp log]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 ntp-b2.nict.go. .INIT.          16 u    -   64    0    0.000    0.000   0.000
 ntp-b3.nict.go. .INIT.          16 u    -   64    0    0.000    0.000   0.000
 ntp-a2.nict.go. .INIT.          16 u    -   64    0    0.000    0.000   0.000
 ntp-a3.nict.go. .INIT.          16 u    -   64    0    0.000    0.000   0.000
*LOCAL(1)        .LOCL.          12 l   31   64    1    0.000    0.000   0.001
```

NTPサーバ名の横に`*`が付いているものは時刻同期が完了した状態，空白は時刻同期中の状態となる．

しばらく経つと，以下のようになる．

```
[root@ntp log]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp-b2.nict.go. .NICT.          1 u   14  128  377    9.551   -0.114   0.069
+ntp-b3.nict.go. .NICT.          1 u   14  128  377    9.551   -0.114   0.069
+ntp-a2.nict.go. .NICT.          1 u   14  128  377    9.551   -0.114   0.069
+ntp-a3.nict.go. .NICT.          1 u   14  128  377    9.551   -0.114   0.069
 LOCAL(1)        .LOCL.          12 l    -   64    0    0.000    0.000   0.000
```

## NTP monlist

ntpd 4.2.7p26より前のバージョンにおいて`ntp.conf`で`disable monitor`を記述しない場合，ntp monlist機能の脆弱性を利用したDDoS攻撃に加担してしまう恐れがある．monlistとはntpサーバの動作をモニタリングするための機能で，クライアントがmonlistのクエリを送るとそのNTPサーバが過去に通信した最大600台のマシンのIPアドレスを返答する．クエリが234バイトであるのに対し，応答パケットのバイトサイズが数十，数百倍となる．加えてNTPは通常UDPを用いて通信を行うことから，容易に送信元IPアドレスを詐称できる．これにより攻撃者が送信元IPアドレスを偽装したmonlistクエリを送信することで大きなサイズのデータを攻撃対象のサーバに送りつけることができ，DoS攻撃が容易に実現できる．

`restrict`のオプションとして`limited`を指定した場合，`disable monitor`は無効になり`monlist`が有効になるようなので注意が必要である．

## 参考

- [NTPサーバー構築(ntpd) - CentOSで自宅サーバー構築](https://centossrv.com/ntp.shtml)
- [Network Time Protocol daemon - ArchWiki](https://wiki.archlinuxjp.org/index.php/Network_Time_Protocol_daemon)
- [NTPを設定してみよう](http://www.peach.ne.jp/ntp.html)
- [ntp_acc(5): Access Control Options - Linux man page](http://linux.die.net/man/5/ntp_acc)
- [AccessRestrictions < Support < NTP](https://support.ntp.org/bin/view/Support/AccessRestrictions)