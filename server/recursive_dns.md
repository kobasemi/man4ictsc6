# Recursive DNS

ここでは，クライアントから名前解決の依頼を受け取り，再帰的に名前解決を行うDNSキャッシュサーバの構築方法についてまとめる．

OS: Ubuntu16.04, Software: BIND

## 必要なツールのインストール

まず，以下のコマンドでBINDをインストールする．

```
$ sudo apt-get install bind9
```

BINDの設定ファイル群は，`/etc/bind/`以下にある．Debian系特有なのか，named.confが機能ごとに複数ファイルに分割されている．

- named.conf
  以下の三つの設定ファイルをインポートするだけの設定ファイル
- named.conf.options
  named.confのoptionsに関するものを記述する
- named.conf.local
  ロギング設定やユーザ自身のゾーン設定などを記述する
- named.conf.default-zones
  ルートゾーンのヒントファイルやlocalhostなどの，デフォルトで入れるべきゾーン設定が記述されている

このうち，ユーザが編集するのは基本的に`named.conf.options`と`named.conf.local`になる．まずは`/etc/bind/named.conf.options`を編集する．

## named.conf.optionsの編集

```
acl intranet {
        10.1.0.0/16;
};
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

		// 自身で回答できない場合に，上流DNSへ問い合わせを転送する
        forwarders { 8.8.8.8; };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation no;

        // 受付ポート番号とIPアドレス
        // キャッシュサーバのため，アクセス制限を設ける．
        /*
           intranet: aclで設定したアドレス空間
           localnets: 自身が所属するアドレス空間
           localhost: 自分自身
        */
        listen-on port 53 { intranet; localnets; localhost; };

        // IPv6は無効にしておく．
        listen-on-v6 { none; };

        // バージョン情報を秘匿する．
        version "unknown";

        // キャッシュサーバとして，外部からのクエリを制限する．
        allow-query { intranet; localnets; localhost; };

        // キャッシュを用いたクエリ応答を制限する．
        allow-query-cache { intranet; localnets; localhost; };

        // キャッシュサーバなので，recursionをyesとする．
        recursion yes;

        // recursionのアクセス元を制限する．
        allow-recursion { intranet; localnets; localhost; };
        
        // Response Rate Limiting，同一IPからの同じようなクエリレスポンスをドロップする．
        // 1秒あたり5クエリまでしか同じようなクエリへのレスポンスを送らない．
        // 5秒間で25クエリまでしか同じようなクエリへのレスポンスを送らない．
        rate-limit {
                responses-per-second 5;
                window 5;
        };
};
```

DNS権威サーバと違い，キャッシュサーバは限定された範囲にのみサービスを公開すべきである．もし仮にキャッシュサーバにアクセス制限を施しておらずグローバルからアクセス可能な状態になっていると，DNS増幅攻撃に加担してしまう恐れがある．

## named.conf.localの編集

```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

logging {
        channel default_debug {
                file "named.log";
                severity dynamic;
        };
};

// ictsc6.local.ゾーンの問い合わせが来た場合は，forwardersに問い合わせを投げる
zone "ictsc6.local" {
        type forward;
        forward only; // 自身で名前解決を行わない
        forwarders {
                192.168.2.21;
                192.168.2.22;
        };
};

// 2.168.192.in-addr.arpa.ゾーンの問い合わせが来た場合は，forwardersに問い合わせを投げる
zone "2.168.192.in-addr.arpa" {
        type forward;
        forward only; // 自身で名前解決を行わない
        forwarders {
                192.168.2.21;
                192.168.2.22;
        };
};
```

`ictsc6.local.`ゾーンや`2.168.192.in-addr.arpa.`ゾーンはルートゾーンから検索できないため，このキャッシュサーバ自身が名前解決するのではなく，このゾーンの権威サーバに問い合わせを投げることで名前解決を行うように設定している．

## 備考

DNS権威サーバのマスターサーバとスレーブサーバと違い，キャッシュサーバの冗長化は基本的に同じ設定のサーバを複数台用意するだけでいい．今回用意したキャッシュサーバについても全く同じ設定を施したサーバを2台用意しているだけである．

## 起動

サーバの設定ができたら，以下のコマンドでサービスを起動する．

```
# systemctl start bind9
```

また，システム起動時に自動的にサービスを起動させるために，以下のコマンドを実行する．

```
# systemctl enable bind9
```

## 参考

- [EZ-NET： 特定のドメインを外部の DNS サーバーへフォワードする (CentOS 5.5): Linux の使い方](http://network.station.ez-net.jp/os/linux/daemon/named/forward/centos/5.5.asp)