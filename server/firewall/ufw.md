# ufw (uncomplicated firewall)

Ubuntu系ディストリビューションで使われている．`iptables`のラッパーで，`iptables`より直感的かつシンプルにファイアウォールを設定出来る．

ufwを有効化する（後述）と，デフォルトでincomingな（内に向かう）通信はドロップされ，outgoingな（外に出て行く）通信は許可される．

初期設定では，ファイアウォールは有効化されていないため，有効化する必要がある．



## 基本的なコマンド（`ufw`）の使い方

### `status`

ファイアウォールの設定を表示

サブコマンドとして，`numbered`と`verbose`が存在する．前者は，ファイアウォールのルールに番号をつけたもので，後者はルールを詳細表示したものである．

```shell
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)

$ sudo ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 22/tcp (v6)                ALLOW IN    Anywhere (v6)

$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)

```



### `allow`

ファイアウォールに許可するルールを追加

```shell
$ sudo ufw allow http
$ sudo ufw allow ssh/tcp
$ sudo ufw allow 22
$ sudo ufw allow 67/udp
$ # IPアドレスとポートを指定して許可
$ sudo ufw allow proto tcp from 192.168.1.1 to any port 22
$ sudo ufw allow from 192.168.100.0/24 to any port 22
```

Tips: `/etc/services`内に記載されているサービスの場合，ポート番号を指定する変わりにサービス名を指定することが出来る（上例における`http`や`ssh`）．



### `deny`

ファイアウォールに拒否するルールを追加

```shell
$ sudo ufw deny 22
$ sudo ufw deny 6379/tcp
$ # IPアドレスとポートを指定して拒否（allowと文法は同様）
$ sudo ufw deny proto tcp from 192.168.1.1 to any port 22
$ sudo ufw deny from 192.168.100.0/24 to any port 22
```



### `delete`

ファイアウォールに追加されているルールを削除

```shell
$ sudo ufw delete deny 22
$ sudo ufw delete deny 22/tcp
```



### `insert`

ファイアウォールに指定した番号でルールを追加

```shell
$ # ルール番号1（一番上）に追加する
$ sudo ufw insert 1 allow 22
```



### `enable`

ファイアウォールの有効化

```shell
$ sudo ufw enable
```



### `disable`

ファイアウォールの無効化

```shell
$ sudo ufw disable
```



### `reload`

ファイアウォールの設定をリロード

```shell
$ sudo ufw reload
```



### `reset`

ファイアウォールの設定をリセット

```shell
$ sudo ufw reset
```



### `logging`

ファイアウォールのログを出力するかどうかを設定

```shell
$ sudo ufw loggin on
$ sudo ufw loggin off
$ sudo ufw loggin full
```

ログの出力先： `/var/log/ufw.log`



## オプション

### `--dry-run`

ファイアウォールに設定することなしに，どういったルールが得られるかを表示

```shell
$ sudo ufw --dry-run deny http
# 略
```



## 高度な使い方

ufwは，iptablesのラッパーであるため，iptablesで可能なことは全て実行出来る．

下記に，ufwで使われる設定ファイルを一覧する．

- `/etc/default/ufw`： デフォルトの設定が書かれている．
- `/etc/ufw/before[6].rules`： ここに書かれたルールは，ufwコマンドで追加したルールの前に評価される．
- `/etc/ufw/after[6].rules`： ここに書かれたルールは，ufwコマンドで追加したルールの後に評価される．
- `/etc/ufw/sysctl.conf`： カーネルのネットワーク設定をする．
- `[/var]/lib/ufw/user[6].rules`： `ufw`コマンドで追加されたルールが書かれている．このファイルを直接編集することは基本的にない．
- `．/etc/ufw/ufw.conf`： 起動時に有効化するかしないか，ログのレベルを設定する．
- `/etc/ufw/after.init`： `ufw`の初期化後に実行するスクリプト
- `/etc/ufw/before.init`： `ufw`の初期化前に実行するスクリプト



## 参考サイト

- [https://wiki.ubuntu.com/UncomplicatedFirewall](https://wiki.ubuntu.com/UncomplicatedFirewall)
- [https://help.ubuntu.com/14.04/serverguide/firewall.html](https://help.ubuntu.com/14.04/serverguide/firewall.html)
- [http://manpages.ubuntu.com/manpages/trusty/en/man8/ufw.8.html](http://manpages.ubuntu.com/manpages/trusty/en/man8/ufw.8.html)
- [http://manpages.ubuntu.com/manpages/trusty/en/man8/ufw-framework.8.html](http://manpages.ubuntu.com/manpages/trusty/en/man8/ufw-framework.8.html)

