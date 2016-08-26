# iptables

ファイアウォールとして，古参の印象．多くのLinuxディストリビューションで使用されている．詳細かつ柔軟にファイアウォールの設定が可能である．

構造的には，テーブルのなかにチェインが存在し，そのチェインに対してルールを設定するといった形になる．

## テーブル

`iptables`には，以下の4つのテーブルが存在する．通常のパケットフィルタリングではfilterテーブルを使用し，NAT機能を使ってルーティングしたい場合にnatテーブルを使用する．

### filterテーブル

このテーブルでは，パケットのフィルタリングの設定を行う．ルールに沿って，パケットの通過を許可するか，それとも遮断するかを決める．

### natテーブル

このテーブルでは，NATに関する設定（DNAT，SNAT，MASQUERADE，REDIRECT）を行う．ルールに沿って，パケットの行き先や戻り先を変更する．

### mangleテーブル

このテーブルは，パケットのTOS (Type of Service)フィールド等といったパケットの改変を行う．

### rawテーブル

このテーブルでは，特定のパケットに対しマークをつけることで，その通信をiptablesで制御しないことを決める．



## チェイン

iptablesにおけるチェインは，各テーブルにおいてフィルタリングをするタイミングを決めるために使われる．チェインは以下の5つが存在し，テーブルごとに使用可能なチェインが分かれる．

### INPUT

インカミング（内向き）に対するチェイン．filterテーブルとmangleテーブルで使用することが出来る．

### OUTPUT

アウトゴーイング（外向き）に対するチェイン．全てのテーブルで使用することが出来る．

### FORWARD

フォワード（転送）に対するチェイン．filterテーブルとmangleテーブルで使用することが出来る．

### PREROUTING

パケットの受信時に送信先アドレスを変更するときに使う．タイミングは，filterテーブルでルールが適用されるより前に行われる．natテーブルとmangleテーブル，rawテーブルで使用することが出来る．

### POSTROUTING

パケットの送信時に送信元アドレスを変更するときに使う．タイミングは，filterテーブルでルールが適用された後に行われる．natテーブルとmangleテーブルで使用することが出来る．

### ユーザ定義チェイン

ユーザがチェインを定義し，テーブルに適応させることが出来る．主に，iptablesの設定を分かりやすくするために使われる．



## ルール

パケットに対して，実際にどのように制御するかを決める部分にあたる．**ACCEPT**や**DROP**，**REJECT**，**DNAT**，**LOG**，**MASQUERADE**などが存在する．



##基本的なコマンド（`iptables`）の使い方 

`iptables`コマンドの書式は以下の通りになる．

```
# iptables [-t テーブル] コマンド [マッチ] [ターゲット/ジャンプ]
```

### `[-t テーブル]`

指定出来るテーブルは，上述した`filter | nat | mangle | raw`の何れかである．なお，デフォルトでfilterテーブルが選択されるため，filterテーブルにルールを追加したい場合は省略することが出来る．それ以外のテーブルに設定するときはテーブルを指定しなければならない．

### `コマンド`

コマンドには，これ以降に記述された部分（マッチやターゲット等）をどのように処理するかを決める．以下に良く使われるコマンドを挙げる．

#### `-A, --append`

指定したチェインの最後尾にルールを追加

```shell
# iptables -A INPUT ...
```

#### `-D, --delete`

指定したチェインからルールを削除．削除するときは，合致するルールの全文を指定するか，ルールナンバーを指定する．

```shell
# iptables -D INPUT --dport 80 -j DROP
# iptables -D INPUT 1
```

#### `-R, --replace`

指定したチェインの指定した行にある既存のルールを変更

```shell
# iptables -R INPUT 1 -s 192.168 ...
```

#### `-I, --insert`

指定したチェインの指定した行に新しいルールを追加

```shell
# iptables --insert INPUT 1 --dport 80 -j ACCEPT
```

#### `-L, --list`

iptablesの設定を一覧表示

```shell
# iptables -L
```

#### `-F, --flush`

指定したチェインの全てのルールを削除

```shell
# iptables -F INPUT
```

#### `-N, --new-chain`

指定したテーブルに新しいチェインを追加

```shell
# iptables -N allowed
```

#### `-X, --delete-chain`

指定したテーブルにあるチェインを削除

```shell
# iptables -X allowed
```

#### `-P, --policy`

指定したチェインに対して，デフォルトのターゲットを設定．チェインに存在するいかなるルールにもマッチしなかったパケットは，ここで指定するデフォルトターゲットによって制御される．

```shell
# iptables -P INPUT DROP
```

### `[マッチ]`

マッチには，その名の通り，どのようなパケットにマッチするかに関する情報を指定する．以下によく使われるものを挙げる．なお，マッチ部では，その指定以外にマッチするといった否定記号を使うことが出来る．否定記号は，`!`で表され，パラメータの直前に配置する．

#### `-p(--protocol) [!] <protocol>` 

特定のプロトコルを指定．プロトコルの種類には，`tcp | udp | icmp`がある．また全てにマッチする`all`を指定することも出来る．

```shell
# iptables -A INPUT -p tcp
# # ICMPパケット以外
# iptables -A INPUT --protocol ! icmp
```

#### `-s(--src, --source) [!] <address>[/mask]`

送信元のIPアドレスを指定．特定のIPアドレスのみを指定するほかに，ネットワーク単位で指定することも出来る．デフォルトでは，全てのIPアドレスにマッチする．

```shell
# iptables -A INPUT -s 192.168.1.1
# iptables -A INPUT --source 192.168.1.0/24
```

#### `-d(--dst, --destination) [!] <address>[/mask]`

送信先のIPアドレスを指定．特定のIPアドレスのみを指定するほかに，ネットワーク単位で指定することも出来る．デフォルトでは，全てのIPアドレスにマッチする．

```shell
# iptables -A INPUT -d 192.168.1.1
# iptables -A INPUT --dst 192.168.1.0/24
```

#### `-i(--in-interface) [!] <interface>[+]`

インカミングな通信に対してネットワークインターフェイスを指定．このマッチは，**INPUT**や**FORWARD**，**PREROUTING**チェインのみで有効である．デフォルトでは，任意の文字または数字からなる文字列を指定したことになり，全てのインターフェイスにマッチする．このとき，`eth+`と指定することで全てのイーサネットネットワークインターフェイスにマッチするといった記述が可能である．

```shell
# iptables -A INPUT -i eth0
# iptables -A INPUT --in-interface ! eth+
```

#### `-o(--out-interface) [!] <interface>[+]`

アウトゴーイングな通信に対してネットワークインターフェイスを指定．このマッチは，**OUTPUT**や**FORWARD**，**PREROUTING**チェインのみで有効である．デフォルトの動作は，上述した`-i, --in-interface`マッチと同様である．

```shell
# iptables -A OUTPUT -o eth0
# iptables -A FORWARD --out-interface ! eth+
```



### 拡張`[マッチ]`

上述したマッチ以外に，さらに詳細なパラメータが指定可能な機能があり，これを拡張マッチという．また，上述したマッチを使えば暗黙的に拡張マッチが使える暗黙的マッチと，拡張マッチを使うことを明示的に指定して使う明示的マッチがある．



暗黙的マッチとして，`-p, --protocol`マッチが挙げられる．このマッチを指定することにより，以下の拡張マッチが使用出来る．以下の拡張マッチを使う場合，`-p, --protocol`マッチより後（右側）に記述しなければならない．

#### `--sport(--source-port) [!] <port>[:port_end]`

送信元ポートを指定．指定するとき，ポートの始まりから終わりまでを指定することも出来る．ポート番号の代わりに，サービス名を指定することが出来るが，その場合は，`/etc/services`にサービス名が記載されている必要がある．また`<port>`を省略し`:<port_end>`のみを指定した場合，`iptables`は自動的にポート番号の始まりを`0`とする．その逆の場合は，`65535`と解釈する．

```shell
# iptables -A INPUT -p tcp --sport 22
# iptables -A INPUT -p tcp --sport ssh
# # TCPのポート番号0から80にマッチ
# iptables -A INPUT -p tcp --sport :80
# # TCPのポート番号80から65535にマッチ
# iptables -A INPUT -p tcp --sport 80:
```

#### `--dport(--destination-port) [!] <port>[:port_end]`

送信先ポートを指定．その他は，`--sport, --source-port`と同様である．

```shell
# iptables -A INPUT -p tcp --dport 22
```

#### `--tcp-flags [!] <test>[,test..] <set>`

TCPプロトコルの場合のみ，TCPフラグを指定．`<test>`で列挙したTCPフラグのうち，`<set>`で指定したフラグのみにマッチする．`<test>`のリストはカンマ区切りで記述し，`<set>`との間に半角スペースを入れて両者を区切る．また通常のTCPフラグのほかに`ALL`と`NONE`を使用出来る．`ALL`は全てのフラグを指定することを意味し，`NONE`はどのフラグも指定しないことを意味する．

```shell
# iptables -p tcp --tcp-flags SYN,FIN,ACK SYN
```

#### `--icmp-type [!] <type>`

ICMPプロトコルの場合のみ，ICMPのタイプを指定．ICMPタイプは，数値および名称で指定出来る．指定可能な名称を調べるには，`# iptables -p icmp --help`を実行すれば良い．

```shell
# iptables -A INPUT -p icmp --icmp-type 8
```



続いて，明示的マッチを使う場合は，`-m(--match) <module>`と記述し有効化したい拡張マッチのモジュールを指定する必要がある．また拡張マッチを使う前（左側）に，予め指定しておく必要がある．使用できるモジュールは，`addrtype`や`comment`，`limit`，`multiport`，`state`，これ以外にも様々なものが存在する．以下では，よく使われる`state`モジュールについて説明する．

#### `--state <STATE>[,STATE..]`

パケットの状態を指定．指定可能な状態は，**INVALID**や**ESTABLISHED**，**NEW**，**RELATED**の4つがある．

```shell
# iptables -A INPUT -m state --state RELATED,ESTABLISHED
```



### `[ターゲット/ジャンプ]`

ターゲットおよびジャンプでは，マッチしたパケットに対して，どのように扱うかを指定する．ターゲットはそのパケットについて許可するかどうか等を指定するのに対し，ジャンプではユーザ定義のチェインを指定する．そのため，同じテーブル内に指定するチェインが存在していなくてはならない．

ここでは，ターゲットの指定についてのみ以下に挙げる．

#### `-j(--jump) ACCEPT`

そのパケットを受け入れる．パケットが受け入れられた場合，同じテーブル内にあるそのルール以降の全てのルールは無視される．

```shell
# iptables -A INPUT -p tcp --dport ssh -j ACCEPT
```

#### `-j(--jump) DROP`

そのパケットを破棄する．このとき，後述する`REJECT`と違い，送信元に対して一切レスポンスを返さない．パケットが破棄された場合，今後いかなるルールも無視される．

```shell
# iptables -A INPUT -p tcp --dport 80 -j DROP
```

#### `-j(--jump) REJECT`

基本的には，`DROP`ターゲットと同じ挙動をするが，パケットをブロックしたときに送信元に対してエラーメッセージを送り返す．またどのようなレスポンスを返すかをオプションで指定することが出来る．

```shell
# iptables -A FORWARD -p TCP --dport 22 -j REJECT --reject-with tcp-reset
```



## コマンド（`iptables`）の主なオプション

#### `-v, --verbose`

結果を詳細表示．`--list`と併用されることが多い．

```shell
# iptables --list --verbose
```



#### `-n, --numeric`

`--list`で表示される設定を，数字を展開して表示．このオプションにより，ホスト名等で表示される変わりに数字が使われる．`--list`コマンドとのみ併用可能である．

```shell
# iptables -L -n
```



#### `--list-numbers`

`--list`で出力される設定に行番号を付与して表示．ルールの挿入時や削除時にルール番号をもとに指定することが出来る．`--list`コマンドとのみ併用可能である．

```shell
# iptables --list-numbers
```



## 使用例

### 現在の`iptables`の設定を表示

```shell
# iptables -vn --line-numbers -L
```

### 確立済みの通信を許可

```shell
# iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

### ローカルホストからの通信を全て許可

```shell
# iptables -A INPUT -i lo -j ACCEPT
```

### icmpパケットを許可

```shell
# iptables -A INPUT -p icmp -j ACCEPT
```

### チェインのデフォルトポリシーを変更

```shell
# iptables -P INPUT DROP
```

### 特定のポートを許可

```shell
# iptables -A INPUT -m state --state NEW -p tcp --dport 80 -j ACCEPT
```

### 特定のIPアドレスからの接続のみ許可

```shell
# iptables -A INPUT -m state --state NEW -p tcp --dport 80 -s 192.168.100.1 -j ACCEPT
```

### 特定のネットワークからの接続のみ許可

```shell
# iptables -A INPUT -m state --state NEW -p tcp --dport 80 -s 192.168.2.0/24 -j ACCEPT
```

### `iptabes`の設定を設定ファイルに反映

```shell
# service iptables save
```



## 設定ファイル

`iptables`で使って設定したファイアウォールの情報は，`/etc/sysconfig/iptables`に格納されている．



## 参考サイト

- [http://www.asahi-net.or.jp/~aa4t-nngk/ipttut/output/ipttut_all.html](http://www.asahi-net.or.jp/~aa4t-nngk/ipttut/output/ipttut_all.html)
- [https://linuxjm.osdn.jp/html/iptables/man8/iptables.8.html](https://linuxjm.osdn.jp/html/iptables/man8/iptables.8.html)
- [http://oxynotes.com/?p=6361](http://oxynotes.com/?p=6361)
- [http://www.asahi-net.or.jp/~aa4t-nngk/iptables.html](http://www.asahi-net.or.jp/~aa4t-nngk/iptables.html)

