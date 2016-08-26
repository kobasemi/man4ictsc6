# Redmineサーバ構築

Ruby on Railsベースのプロジェクト管理WebアプリケーションであるRedmineを構築する．

OS: CentOS7

## インストールする前に

### SELinuxの無効化

以下のコマンドを実行して，現在のSELinuxの状態を把握する．

```
# getenforce
```

実行結果として`Enforcing`が表示された場合，SELinuxが有効になっている．この場合，`/etc/sysconfig/selinux`の`SELINUX=`に`disabled`を設定する．

```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

変更したら，システムを再起動して再度`getenforce`でSELinuxの状態を確認し，`Disabled`になっているか確認する．

### firewalldでHTTP通信を許可

`/etc/firewalld/zones/public.xml`を編集し，`zone`タグ内に以下のものを追加する．

```
<service name="http"/>
```

追加できたら，firewalldサービスをリロードする．

```
# systemctl reload firewalld
```

## Redmineのインストール

SELinuxの無効化とfirewalldの設定が済んだら，Redmineのインストールに移る．

### Redmineを動作させるために必要な，関連パッケージのインストール

Redmineをインストールする前に，関連するパッケージのインストールを済ませておく．

- Development Tools

  ```
  # yum groupinstall "Development Tools"
  ```

- RubyとPassengerのビルドに必要なヘッダファイルなど

  ```
  # yum install openssl-devel readline-devel zlib-devel curl-devel libyaml-devel libffi-devel
  ```

- PostgreSQLとヘッダファイル

  ```
  # yum install postgresql-server postgresql-devel
  ```

- Apacheとヘッダファイル

  ```
  # yum install httpd httpd-devel
  ```

- ImageMagickとヘッダファイル，日本語フォントのインストール
  ガントチャートをPNGでエクスポートする機能，添付ファイルのサムネイル画像を作成するのに必要

  ```
  # yum install ImageMagick ImageMagick-devel ipa-pgothic-fonts
  ```

- Rubyのインストール
  以下のコマンドで，公式のダウンロードページからRuby2.2.5のtarballをダウンロードする．

  ```
  # curl -L -O https://cache.ruby-lang.org/pub/ruby/2.2/ruby-2.2.5.tar.gz
  ```

  ダウンロードできたら以下のコマンドで展開し，Rubyのビルドとインストールを行う．

  ```
  # tar zxvf ruby-2.2.5.tar.gz
  # cd ruby-2.2.5
  # ./configure --disable-install-doc
  # make
  # make install
  # cd ..
  ```

- Rubyのパッケージ管理ツールであるbundlerをインストールする．

  ```
  # gem install bundler --no-rdoc --no-ri
  ```

### PostgreSQLの設定

先ほどインストールしたPostgreSQLに，Redmineのための設定を追加する．まず，PostgreSQLのデータベースクラスタ（データベースのまとまり，ディスク上のデータベース格納領域のこと）を以下のコマンドで初期化する．

```
# postgresql-setup initdb
```

次に，`/var/lib/pgsql/data/pg_hba.conf`の下部，`Put your actual configuration here`の部分に以下を追加する．

```
host	redmine		redmine		127.0.0.1/32		md5
host	redmine		redmine		::1/128				md5
```

編集できたら，以下のコマンドでPostgreSQLの起動と，システム起動時に自動起動するよう設定する．

```
# systemctl start postgresql
# systemctl enable postgresql
```

PostgreSQLが起動したら，以下のコマンドでデータベースへのアクセスが可能なpostgresユーザにスイッチし，Redmine用のPostgreSQLユーザを作成する．

```
# su postgres
$ createuser -P redmine
```

`-P`オプションを指定することで，このユーザにパスワードを設定する．

ユーザが作成されたことを確認するためには，`psql`コマンドでPostgreSQLに接続し，以下のクエリを実行する．

```
select usename from pg_user;
```

`usename`は打ち間違いではないのでこのまま実行すること．このクエリにより，登録されているユーザ名が一覧で表示される．

> 上記の例ではPostgreSQLで用意された`createuser`コマンドを用いてユーザを作成したが，別の方法として`psql`コマンドでPostgreSQLに接続し，以下のクエリを実行することでもユーザを作成できる．
>
> ```
> create user redmine with password 'password';
> ```

ユーザを作成できたら，次にRedmine用のデータベースを作成する．まずpostgresユーザにスイッチしてからデータベースを作成する．

```
# su postgres
$ createdb -E UTF-8 -l ja_JP.UTF-8 -O redmine -T template0 redmine
```

- `-E`：エンコードを指定
- `-l`：ロケールを指定
- `-O`：データベースの所有者を指定
- `-T`：このデータベースの構築に使用するテンプレートデータベースを指定

> 上記の例ではPostgreSQLで用意された`createdb`コマンドを用いてデータベースを作成したが，別の方法として`psql`コマンドでPostgreSQLに接続し，以下のクエリを実行することでもデータベースを作成できる．
>
> ```
> create database redmine encoding 'UTF8' lc_collate 'ja_JP.UTF-8' lc_ctype 'ja_JP.UTF-8' owner redmine template template0;
> ```

### Redmineのインストール

いよいよRedmineのインストールに入る．

#### Redmine本体のダウンロード

Redmine本体のtarballを公式のダウンロードページから以下のコマンドでダウンロードする．

```
# curl -L -O http://www.redmine.org/releases/redmine-3.3.0.tar.gz
```

そして，以下のコマンドで`/var/lib/`以下にredmineのtarballを展開する．

```
# mkdir /var/lib/redmine
# tar zxvf redmine-3.3.0.tar.gz -C /var/lib/redmine --strip-components 1
```

> tarballをダウンロードして展開，ではなくSubversionやMercurial，Gitのコマンドを用い，クローンすることでダウンロードするという方法も可能である．例えばSubversionのリポジトリからダウンロードするには以下のコマンドを実行する．
>
> ```
> # svn co https://svn.redmine.org/redmine/branches/3.3-stable /var/lib/redmine
> ```

#### データベース接続用設定ファイルの作成

ダウンロードできたら，Redmine側でデータベースに接続するための設定ファイルを作成する．`/var/lib/redmine/config/database.yml`を作成し，以下の内容を記述する．

```
production:
  adapter: postgresql
  database: redmine
  host: localhost
  username: redmine
  password: "PostgreSQLにredmineユーザを追加した際に指定したパスワード"
  encoding: utf8
```

#### Redmine設定ファイルの作成

次に，Redmine全般の設定を行うファイルを作成する．`/var/lib/redmine/config/configuration.yml`を作成し，以下の内容を記述する．

```
production:
  rmagick_font_path: /usr/share/fonts/ipa-pgothic/ipagp.ttf
```

#### 依存Gemパッケージのインストール

設定ファイルを作成できたら以下のコマンドを実行して，`/var/lib/redmine`に移動しRedmineの依存gemパッケージをインストールする．

```
# cd /var/lib/redmine
# bundle install --without development test --path vendor/bundle
```

#### セッション改ざん防止用トークンの作成

依存gemをインストールできたら，Redmineのセッション改ざん防止用のトークンを以下のコマンドで作成する．

```
bundle exec rake generate_secret_token
```

#### Redmine用データベースの準備

次に，Redmine用データベースに以下のコマンドでテーブルを作成する．

```
RAILS_ENV=production bundle exec rake db:migrate
```

作成したテーブルにデフォルトのデータを投入する．

```
RAILS_ENV=production REDMINE_LANG=ja bundle exec rake redmine:load_default_data
```

#### Passengerのインストール

Apache上でRailsアプリケーションを動かすために使われるPhusion Passengerを以下のコマンドでインストールする．

```
# gem install passenger --no-rdoc --no-ri
```

Passengerをインストールできたら，以下のコマンドを実行してApache用のモジュールをインストールする．

```
# passenger-install-apache2-module --auto
```

以下のコマンドを実行すると，ApacheでPassengerを用いるための設定方法を確認できる．

```
# passenger-install-apache2-module --snippet
```

実行すると，以下のような出力を得られる．

```
LoadModule passenger_module /usr/local/lib/ruby/gems/2.2.0/gems/passenger-5.0.30/buildout/apache2/mod_passenger.so
<IfModule mod_passenger.c>
  PassengerRoot /usr/local/lib/ruby/gems/2.2.0/gems/passenger-5.0.30
  PassengerDefaultRuby /usr/local/bin/ruby
</IfModule>
```

これに従いつつ，`/etc/httpd/conf.d/redmine.conf`を作成して以下の内容を記述する．

```
# Allow to access Redmine resource files
<Directory "/var/lib/redmine/public">
  Require all granted
</Directory>

# Basic configuration of Passenger module
LoadModule passenger_module /usr/local/lib/ruby/gems/2.2.0/gems/passenger-5.0.30/buildout/apache2/mod_passenger.so
<IfModule mod_passenger.c>
  PassengerRoot /usr/local/lib/ruby/gems/2.2.0/gems/passenger-5.0.30
  PassengerDefaultRuby /usr/local/bin/ruby
</IfModule>

# Remove HTTP headers added by Passenger
Header always unset "X-Powered-By"
Header always unset "X-Runtime"

# Tuning Passenger：https://www.phusionpassenger.com/library/config/apache/reference/

# Disable user friendry error page
PassengerFriendlyErrorPages off

# Minimize process spawning：https://www.phusionpassenger.com/library/config/apache/optimization/
# (TOTAL_RAM * 0.75) / RAM_PER_PROCESS = (2048 * 0.75) / 150 = 10.24
PassengerMaxPoolSize 10
PassengerMinInstances 10

# Reduce compatibility with other Apache modules (mod_rewrite, mod_autoindex, ...)
# Passenger will be a little faster
PassengerHighPerformance on

# Using a prefork copy-on-write mechanism to spawn applications
PassengerSpawnMethod smart
```

作成できたら，ApacheからRedmineのディレクトリを読み書きできるように権限を変更する．

```
# chown -R apache:apache /var/lib/redmine
```

#### Apacheの設定（ドキュメントルートの指定）

`/etc/httpd/conf/httpd.conf`を開き，`ServerName`と`DocumentRoot`の部分を以下のように編集する．

```
ServerName redmine.ictsc6.local:80
DocumentRoot "/var/lib/redmine/public"
```

## Apacheの起動

起動する前に，以下のコマンドを実行して設定ファイルをチェックする．

```
# service httpd configtest
```

`Syntax OK`と返ってきたら以下のコマンドを実行して，Apacheの起動と，システム起動時に自動起動する設定を行う．

```
# systemctl start httpd
# systemctl enable httpd
```

## うまく動かない場合は…

まず，Apacheのエラーログを確認する．エラーログのパスは`/var/log/httpd/error_log`．また，Redmineのエラーログは`/var/lib/redmine/log/production.log`にある．

` Message from application: FATAL:  Ident authentication failed for user "redmine"`というようなエラーログがあれば，`/var/lib/pgsql/data/pg_hba.conf`の記述に誤りがある．`Ident`はシステムローカルのユーザ名とPostgresSQLのユーザ名が一致していれば受け入れる設定である．

今回Redmineサーバを構築する段階でこのエラーに遭遇した．PostgreSQLの`redmine`ユーザの認証は`Ident`ではなく`md5`，つまりパスワード認証を設定していたつもりだったので全く意図しないエラーだった．その時は`/var/lib/pgsql/data/pg_hba.conf`を以下のように設定していた．

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
host    redmine         redmine         127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 ident
host    redmine         redmine         ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            ident
#host    replication     postgres        ::1/128                 ident
```

これを，`host all all 127.0.0.1/32 ident`と`host redmine redmine 127.0.0.1/32 md5`，`host all all ::1/128 ident`と`host redmine redmine ::1/128 md5`のそれぞれの順序を以下のように入れ替えた．

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    redmine         redmine         127.0.0.1/32            md5
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    redmine         redmine         ::1/128                 md5
host    all             all             ::1/128                 ident
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            ident
#host    replication     postgres        ::1/128                 ident
```

その上でPostgreSQLを再起動し，`Redmine用データベースの準備`を再度行うことで，このエラーを解消できた．`/var/lib/pgsql/data/pg_hba.conf`を編集する際は，ルールの**順序**に気を配る必要がある．

## 参考

- [Redmine 3.2をCentOS 7.1にインストールする手順 | Redmine.JP Blog](http://blog.redmine.jp/articles/3_2/install/centos/)
- [CentOS7にRedmineを構築する - write ahead log](http://twinbird-htn.hatenablog.com/entry/2016/02/26/203314)
- [Configuration reference - Apache - Passenger Library](https://www.phusionpassenger.com/library/config/apache/reference/#passengerfriendlyerrorpages)
- [CentOS への PostgreSQL 導入 - clock-up-blog](http://blog.clock-up.jp/entry/2014/12/30/centos-postgresql)
- [Redmine postgresql FATAL: Ident authentication failed for user "redmine" - Stack Overflow](http://stackoverflow.com/questions/9942963/redmine-postgresql-fatal-ident-authentication-failed-for-user-redmine)
- [ログイン — Redmine.JP](http://redmine.jp/tech_note/first-step/admin/login/)
- [adminユーザのパスワードの変更 — Redmine.JP](http://redmine.jp/tech_note/first-step/admin/admin-user-password/)