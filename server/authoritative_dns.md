# Authoritative DNS

ここでは，マスターサーバとスレーブサーバをそれぞれ一台づつ用意し，冗長化させたDNS権威サーバについてまとめる．

## 構築に入る前に…

DNSサーバと一口に言っても，機能によって二つに大別される．まず，DNSキャッシュサーバ（DNSフルサービスリゾルバ）がある．これはクライアント側でDNS設定する際に指定されるサーバで，ドメイン名をルートから辿って名前解決を行う機能を持つサーバのことをいう（クライアントからの問い合わせに答えるのが再帰検索，クライアントからの問い合わせに対してルートサーバから順に辿っていくのが反復検索）．このサーバには通常キャッシュ機能があり，以前問い合わせがあったドメイン名についてキャッシュを用いて高速に名前解決できる仕組みになっている．ドメイン名の管理はしていない，つまりドメイン名とIPアドレスの対応付けを変更する権限を持っていないことからNon-authorized（無権限）サーバなどとも言われる．

先述したように，DNSキャッシュサーバは名前解決を問い合わせるだけで，これ自身がドメイン名の管理をしているわけではない．ドメイン名を管理し，名前解決に必要な情報を提供するDNSサーバのことをDNS権威サーバという．このサーバは基本的に全世界に公開され，自らの管理するゾーン内のドメイン名とIPアドレスの対応付けを自由に変更する権限を持っていることから権威サーバと呼ばれる．自らの持つゾーンの管理を下位の権威サーバに委任することが可能であり，これによってDNSの分散管理が実現できている．また，権威サーバに万が一障害が発生した際にもサービスを継続的に提供できるように，権威を持たないサーバを用意し，権威サーバのゾーン情報を転送する（ゾーン転送）ことで冗長性を確保する構成が一般的である．この場合の権威を持つサーバのことをDNSマスターサーバ，権威を持たないサーバのことをDNSスレーブサーバと呼んだりし，ゾーン情報を持つことからこれらをまとめてDNSコンテンツサーバと呼ぶ．

## DNS権威サーバ（マスターサーバ）

OS: CentOS7, Software: BIND

### 必要なツールのインストール

まず，以下のコマンドでBINDとchroot環境でBINDを動かすためのツールを入れる．

```
# yum install bind bind-chroot
```

### BIND設定（named.conf編集）

BINDの設定ファイルである`named.conf`を編集する．CentOS7では`/etc/named.conf`にある．

デフォルトの`named.conf`を以下に載せる．

```conf
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        listen-on port 53 { 127.0.0.1; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { localhost; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

これを，以下のように編集した．

```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        // 受付ポート番号とIPアドレス
        // 権威サーバのため，他ホストからアクセス可能なように設定する．
        listen-on port 53 { any; };

        // IPv6は無効にしておく．
        listen-on-v6 { none; };

        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";

        // バージョン情報を秘匿する
        version "unknown";

        // 権威サーバとして，外部からのクエリを受け付けるように設定する．
        allow-query     { any; };
        // キャッシュを用いた回答をしない
        allow-query-cache       { none; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        // 上の説明にもあるように，権威サーバとして設定するのでrecursionをnoとする．
        // これにより，このサーバが管理するゾーン以外は解決しない．（再帰検索不可）
        recursion no;

        /*
           DNSSECを用いることで，DNSレコードの改ざんを防止できる．
           権威サーバ側でレコードに署名し，キャッシュサーバで署名を検証することで応答の真正性を確保する．
           本来はこの権威サーバでも登録するレコードに署名をし，上位ゾーンとの信頼の連鎖を構築するためのKSK
           を上位ゾーンにDSレコードとして登録してもらう必要がある．
        */
        dnssec-enable no;
        dnssec-validation no;
        dnssec-lookaside no;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        // Response Rate Limiting，同一IPからの同じようなクエリレスポンスをドロップする．
        // 1秒あたり5クエリまでしか同じようなクエリへのレスポンスを送らない．
        // 5秒間で25クエリまでしか同じようなクエリへのレスポンスを送らない．
        rate-limit {
                responses-per-second 5;
                window 5;
        };
};


logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

// 正引き用ゾーンの指定
zone "ictsc6.local" {
        type master;
        // 上で設定した directory からの相対パス
        file "ictsc6.local.db";
        also-notify     { 192.168.2.22; };
};

// 逆引き用ゾーンの指定
zone "2.168.192.in-addr.arpa" {
        type master;
        // 上で設定した directory からの相対パス
        file "2.168.192.in-addr.arpa.db";
        also-notify     { 192.168.2.22; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

#### options設定

`options`内には，BIND全体の設定を記述する．ここで構築するのはDNSの**権威**サーバである．権威サーバはその性質上，外部ネットワークから誰でもアクセスできなければならない．これを実現するために，以下のような設定をする．

```
listen-on port 53 { any; };
allow-query { any; };
```

外部ネットワークから誰でもアクセス可能になるため，BINDのバージョン情報を秘匿してクラックされるリスクを下げることが重要となる．以下の設定でバージョン情報を指定した文字列に置換できる．

```
version "unknown";
```

権威サーバには自身が管理するゾーン情報がある．キャッシュサーバから来た問い合わせにはこのゾーン情報をもとにした返答のみをさせたい．つまりキャッシュを用いた返答を避けたい．この場合以下のような設定をする．

```
allow-query-cache { none; };
```

権威サーバを構築しているため，再帰検索機能（ルートから辿って名前解決を行う機能）は不要である．自身が管理するゾーン情報においてのみ返答させ，自身が管理しないゾーンについては返答を拒否させるには以下の設定をする．

```
recursion no;
```

権威サーバは外部ネットワークから誰でもアクセス可能である．DNSはUDPプロトコルベースで動作するサービスであり，アクセス元IPアドレスの詐称が可能である．加えて問い合わせに比較して応答のサイズが必ず大きくなることからDNSリフレクター攻撃によるDDoSが頻発している．

これを低減させるためには，同一アクセス元からの，瞬間的に大量の同じような問い合わせに対して応答をドロップするような設定が有効である．これはDNS RRL（Response Rate Limiting）という仕組みで実現可能で，以下のように設定すればいい．

```
rate-limit {
	responses-per-second 5;
	window 5;
};
```

#### zone設定

ゾーンとは，広大なドメイン名空間の中で自組織が運用する権威サーバが担当する部分的なドメイン名空間のことである．ここでは，ICTSC6用に`ictsc6.local.`と`2.168.192.in-addr.arpa.`という二つのゾーンを用意し，設定する．

まず，`named.conf`に対して，以下の記述を追加する．

```
zone "ictsc6.local" {
	type master;
	file "ictsc6.local.db";
	also-notify { 192.168.2.22; };
};

zone "2.168.192.in-addr.arpa" {
	type master;
	file "2.168.192.in-addr.arpa.db";
	also-notify { 192.168.2.22; };
};
```

`type master;`は，この権威サーバをスレーブサーバでなくマスターサーバとして機能させるという設定である．`file`はそのゾーンについてのレコードを記述したゾーンファイルのパスを記述する．上記の場合，先に設定した`directory`からの相対パスとなる．もちろん絶対パスでの記述も可能である．`also-notify`は，ゾーンファイルの修正があり，ゾーンファイル中のシリアル番号が増えた場合にここで指定したアドレスへ通知するという設定となる．ここにはスレーブサーバのIPアドレスを記述する．

以上でBIND自体の設定が完了した．次に`zone`で指定したゾーンファイルを作成する．

### ゾーンファイルの作成

これから作成するのは，正引き（ドメイン名 -> IPアドレス）用のゾーンファイルと逆引き（IPアドレス -> ドメイン名）用のゾーンファイルである．作成する場所は，`named.conf`の`directory`で指定した，`/var/named/`直下となる．

#### 正引きゾーンファイル作成

`/var/named/ictsc6.local.db`を以下の内容で作成する．

```
$TTL    86400   ; キャッシュの有効期限，短すぎても長すぎてもいけない．
@       IN      SOA     master.ictsc6.local.    root.ictsc6.local. (
        2016072801      ; serial
        43200           ; refresh 12 hours
        7200            ; retry 2 hours
        1209600         ; expire 2 weeks
        604800          ; negative cache ttl, 存在しないドメイン名のキャッシュの有効期限
)
        IN      NS      master.ictsc6.local.

master          IN      A       192.168.2.21
slave           IN      A       192.168.2.22
samba           IN      A       192.168.2.38
dhcp            IN      A       192.168.2.67
primary         IN      A       192.168.2.80
secondary       IN      A       192.168.2.81
```

`$TTL`ではキャッシュの有効期限を設定する．レコードを変更しても，キャッシュサーバではこのTTLが切れるまでキャッシュが残り続ける．`SOA`とはStart Of Authorityの略語で，$TTLの後ろに書かなければならない．SOAの後ろにはまず`MNAME`，ゾーンファイルの元データを持つネームサーバのFQDNを記述する．その後ろには`RNAME`，このドメインの管理者のメールアドレスを記述する．なお，`@`記号はゾーン名自身を表す（正確には直近の`$ORIGIN`で指定されたゾーン名，指定がない場合はnamed.confで指定されたゾーン名となる）ので使ってはいけない．`@`記号を`.`に置き換えて記述する．

`()`内だが，まず始めに`SERIAL`を書く．これはゾーンファイルのバージョンを表すもので，一般的には`YYYYMMDDnn`，例えばゾーンファイルが2016年7月30日に第2版へ更新されたなら`2016073002`というように書く．次は`REFRESH`，スレーブサーバがゾーン更新の問い合わせをする間隔を指定する．その次は`RETRY`，`REFRESH`が何らかの原因で機能しなかった，ゾーン情報の更新ができなかった場合にここで指定した時間後に再試行する．次は`EXPIRE`，何らかの原因でゾーン情報のリフレッシュができない状態が続いた場合に，スレーブサーバが持っているデータをどれだけ利用できるかを指定する．SOAレコードの最後の設定項目である`MINIMUM`は存在しないドメイン名であるという情報をキャッシュする時間を指定する．

SOAレコードの次にある`NS`レコードだが，これはゾーンに対する権威を持つサーバを指定するものである．権威サーバを複数指定することも可能で，その場合は以下のようにする．

```
IN	NS	ns1.ictsc6.local.
IN	NS	ns2.ictsc6.local.
```

最後に`A`レコードだが，これはドメイン名とIPアドレスの関連付けをする．ドメイン名一つに対して複数のIPアドレスを関連付けることもでき（DNSラウンドロビン），また逆に一つのIPアドレスに対して複数のドメイン名を関連付けることもできる．

#### 逆引きゾーンファイル作成

`/var/named/2.168.192.in-addr.arpa.db`を以下の内容で作成する．

```
$TTL    86400   ; キャッシュの有効期限，短すぎても長すぎてもいけない．
@       IN      SOA     master.ictsc6.local.    root.ictsc6.local. (
        2016072801      ; serial
        43200           ; refresh 12 hours
        7200            ; retry 2 hours
        1209600         ; expire 2 weeks
        604800          ; negative caache ttl, 存在しないドメイン名のキャッシュの有効期限
)
        IN      NS      master.ictsc6.local.

21      IN      PTR     master.ictsc6.local.
22      IN      PTR     slave.ictsc6.local.
38      IN      PTR     samba.ictsc6.local.
67      IN      PTR     dhcp.ictsc6.local.
80      IN      PTR     primary.ictsc6.local.
81      IN      PTR     secondary.ictsc6.local.
```

`$TTL`や`SOA`，`NS`は正引きゾーンファイルと同じなので説明を省く．逆引きゾーンファイルでは，`PTR`レコードを用いてIPアドレスとドメイン名の関連付けをする．`A`レコードでは1対多の設定ができたが，`PTR`レコードでは基本的に一つのIPアドレスに対しては一つのドメイン名を設定する．

### 設定ファイルのチェックと，起動

DNSサーバを起動させる前に，設定ファイルのチェックをおすすめする．`bind`をインストールした際についてくる`named-checkconf`コマンドと`named-checkzone`コマンドで設定ファイルの記述に明らかな誤りがないかを確認できる．

まず，`named.conf`のチェックは以下のように行う．

```
# named-checkconf /etc/named.conf
```

誤りがなければ何も表示されない．

次に，ゾーン設定のチェックを以下のように行う．

```
# named-checkzone ictsc6.local. /var/named/ictsc6.local.db
```

誤りがなければ，OKというレスポンスが返ってくる．

以上二つのチェックで誤りが確認されなければ，以下のコマンドでDNSを起動する．

```
# systemctl start named-chroot
```

起動後，動作ログは`named.conf`の`logging`で指定したファイルと，以下のコマンドから確認できる．

```
# journalctl -f -u named-chroot
```

また，以下のコマンドでサーバの起動時に自動的にDNSが起動するように設定できる．

```
# systemctl enable named-chroot
```

### ゾーンデータの更新

ゾーンデータを更新するときは，ゾーンファイルを編集し，シリアル番号を増やす．編集後，以下のコマンドを実行して更新を反映させる．

```
# rndc reload
```



## DNS権威サーバ（スレーブサーバ）

OS: CentOS7, Software: BIND

### 必要なツールのインストール

まず，以下のコマンドでBINDとchroot環境でBINDを動かすためのツールを入れる．

```
# yum install bind bind-chroot
```

### BIND設定（named.conf編集）

BINDの設定ファイルである`named.conf`を以下のように編集する．

```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        // 受付ポート番号とIPアドレス
        // 権威サーバのため，他ホストからアクセス可能なように設定する．
        listen-on port 53 { any; };

        // IPv6は無効にしておく．
        listen-on-v6 port 53 { none; };

        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";

        // バージョン情報を秘匿する
        version "unknown";

        // 権威サーバとして，外部からのクエリを受け付けるように設定する．
        allow-query     { any; };
        // キャッシュを用いた回答をしない
        allow-query-cache       { none; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        // 上の説明にもあるように，権威サーバとして設定するのでrecursionをnoとする．
        // これにより，このサーバが管理するゾーン以外は解決しない（再帰検索不可）．
        recursion no;

        /*
           DNSSECを用いることで，DNSレコードの改ざんを防止できる．
           権威サーバ側でレコードに署名し，キャッシュサーバで署名を検証することで応答の真正性を確保する．
           本来はこの権威サーバでも登録するレコードに署名をし，上位ゾーンとの信頼の連鎖を構築するためのKSK
           を上位ゾーンにDSレコードとして登録してもらう必要がある．
        */
        dnssec-enable no;
        dnssec-validation no;
        dnssec-lookaside no;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        // Response Rate Limiting, 同一IPからの同じようなクエリレスポンスをドロップする．
        // 1秒あたり5クエリまでしか同じようなクエリへのレスポンスを送らない．
        // 5秒間で25クエリまでしか同じようなクエリへのレスポンスを送らない．
        rate-limit {
                responses-per-second 5;
                window 5;
        };
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

// 正引き用ゾーンの指定（スレーブ）
zone "ictsc6.local" {
        type slave;
        // 上で設定した directory からの相対パス
        file "slaves/ictsc6.local.db";
        masters { 192.168.2.21; };
};

zone "2.168.192.in-addr.arpa" {
        type slave;
        // 上で設定した directory からの相対パス
        file "slaves/2.168.192.in-addr.arpa.db";
        masters { 192.168.2.21; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

#### options設定

`options`内の設定については，マスターサーバと同じであるため説明を省略する．

#### zone設定

マスターサーバと同様に，`ictsc6.local.`と`2.168.192.in-addr.arpa.`という二つのゾーンを用意し，設定する．

`named.conf`に対して，以下の記述を追加する．

```
zone "ictsc6.local" {
        type slave;
        file "slaves/ictsc6.local.db";
        masters { 192.168.2.21; };
};

zone "2.168.192.in-addr.arpa" {
        type slave;
        file "slaves/2.168.192.in-addr.arpa.db";
        masters { 192.168.2.21; };
};
```

`type slave;`と記述し，このサーバをマスターサーバでなくスレーブサーバとして機能させるという設定をしている．`file`では先に設定した`directory`の下にある`slaves/`ディレクトリの下，デフォルトでは`/var/named/slaves/`の下にマスターサーバから受け取ったゾーンデータを保存するように設定している．最後に`masters`でマスターサーバのIPアドレスを指定している．

### 設定ファイルのチェックと，起動

DNSサーバを起動させる前に，設定ファイルのチェックをおすすめする．スレーブサーバではゾーンファイルがないので，`named-checkconf`を用いて`named.conf`のチェックのみを行う．

```
# named-checkconf /etc/named.conf
```

誤りがなければ何も表示されない．

チェックで誤りが確認されなければ，以下のコマンドでDNSを起動する．

```
# systemctl start named-chroot
```

起動後，動作ログは`named.conf`の`logging`で指定したファイルと，以下のコマンドから確認できる．

```
# journalctl -f -u named-chroot
```

また，以下のコマンドでサーバの起動時に自動的にDNSが起動するように設定できる．

```
# systemctl enable named-chroot
```

ログを確認して，ゾーン転送が上手くいっているかを確認する．ゾーン転送が成功していれば，`/var/named/slaves/`の下にゾーンデータが生成されているはずだ．

### ゾーン情報の更新

マスターサーバからNOTIFYメッセージが届かないなどの理由でゾーン転送が行われない場合，スレーブサーバ上で以下のコマンドを実行することですぐにゾーン転送を行える．

```
# rndc refresh ictsc6.local
```

`ictsc6.local`の部分には，ゾーン転送を希望するゾーン名を指定する．ただし，この場合でもマスターサーバのゾーンのシリアル番号がスレーブサーバよりも増えていない場合はゾーン転送が開始されないため，注意が必要である．

## 参考

- [設定ガイド：オープンリゾルバー機能を停止するには【BIND編】](https://jprs.jp/tech/notice/2013-04-18-fixing-bind-openresolver.html)
- [DNSサーバー構築(BIND) - CentOSで自宅サーバー構築](https://centossrv.com/bind.shtml)
- [内部向けDNS構築・解説 CentOS7.1 - Qiita](http://qiita.com/ToraLin/items/ae251b187d18de7684eb)
- [BINDの設定方法（オープンリゾルバ対策含む）](http://hikaku-server.com/linux/entry306.html)
- [BINDによる内部向けDNSの構築](http://www.tooyama.org/bind-lan.html)
- [強いBIND DNSサーバを構築する　第二回　named.confの基本設定 | ユーロテック情報システム販売株式会社](http://www.eis.co.jp/bind9_src_build_2/)
- [domain name system - Bind DNS rate-limit and values for responses-per-second and window - Server Fault](http://serverfault.com/questions/490245/bind-dns-rate-limit-and-values-for-responses-per-second-and-window)
- [ CentOS 7 のBINDを設定する (プライマリ コンテンツサーバー) ](https://www.ipentec.com/document/document.aspx?page=linux-centos-7-bind-configuration)