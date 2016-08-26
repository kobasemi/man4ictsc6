# LDAPサーバの構築

OS: CentOS

以下のコマンドでOpenLDAPサーバ構築に必要なパッケージをインストールする．

```
# yum install openldap-servers openldap-clients
```

なお，`openldap-clients`をインストールしているのは，LDAPサーバにエントリを登録する際に必要となる`ldapadd`コマンドなどが`openldap-clients`に含まれているためである．

## LDAPサーバの設定

移行の設定は，OLC（On-Line Configuration）を用いている（slapd.confを用いた設定はもう古いようだ）．

### サンプル設定ファイルのコピー，OpenLDAPの起動

以下のコマンドを実行し，サンプルの設定ファイルをコピーする．

```
# cp -a /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
# chown ldap:ldap /var/lib/ldap/DB_CONFIG
```

設定ファイルをコピーできたら，以下のコマンドでOpenLDAPサーバを起動し，システム起動時に自動起動するように設定する．

```
# systemctl start slapd
# systemctl enable slapd
```

### OpenLDAPの管理者パスワードを生成し，設定

以下のコマンドで管理者パスワードのSSHA（ソルト付きSHA）を生成する．

```
# slappasswd
```

入力指示に従ってパスワードを入力すると，以下のような表示でSSHAを得られる．

```
{SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

これを用いて，以下のような管理者パスワード設定用の.ldifファイル`changerootpw.ldif`を作成する．

```
dn: olcDataBase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

そして，以下のコマンドでこの.ldifファイルを用いてパスワードを設定する．

```
ldapadd -Y EXTERNAL -H ldapi:/// -f changerootpw.ldif
```

正しく処理が進んでいれば，以下のようなメッセージが返ってくる．

```
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={0}config,cn=config"
```

### OpenLDAPにschemaを追加

後々ObjectClassを指定する際などで必要になってくるschemaをここで追加しておく．具体的には以下のコマンドを実行する．

```
# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
```

`shadowLastChange`などを用いるのに必要となる．

### LDAPサーバのドメイン設定

まず，以下のコマンドでディレクトリマネージャのSSHAを生成する．

```
# slappasswd
```

そして，DNを変更するために以下の.ldifファイル`changedn.ldif`を作成する．

```
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
  read by dn.base="cn=Manager,dc=ictsc6,dc=local" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ictsc6,dc=local

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=ictsc6,dc=local

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=Manager,dc=ictsc6,dc=local" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=ictsc6,dc=local" write by * read
```

作成できたら，以下のコマンドを実行してこの.ldifファイルに書かれた変更を反映する．

```
# ldapmodify -Y EXTERNAL -H ldapi:/// -f changedn.ldif
```

次に，ベースドメインを登録するために，以下の.ldifファイル`basedomain.ldif`を作成する．

```
dn: dc=ictsc6,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: ICTSC6
dc: ictsc6

dn: cn=Manager,dc=ictsc6,dc=local
objectClass: organizationalRole
cn: Manager
description: Directory Manager

dn: ou=People,dc=ictsc6,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=ictsc6,dc=local
objectClass: organizationalUnit
ou: Group
```

作成できたら，以下のコマンドでこれを追加する．

```
# ldapadd -x -D cn=Manager,dc=ictsc6,dc=local -W -f basedomain.ldif
```

## ユーザアカウントの追加

あらかじめ`slappasswd`でパスワードのSSHAを得た上で，以下のような.ldifファイル`taro.ldif`を作成する．

```
dn: uid=taro,ou=People,dc=ictsc6,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Taro
sn: Yamada
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
loginShell: /bin/bash
uidNumber: 1000
gidNumber: 1000
homeDirectory: /home/taro

dn: cn=taro,ou=Group,dc=ictsc6,dc=local
objectClass: posixGroup
cn: Taro
gidNumber: 1000
memberUid: taro
```

作成できたら，以下のコマンドで反映する．

```
# ldapadd -x -D cn=Manager,dc=ictsc6,dc=local -W -f taro.ldif
```

## 参考

- [CentOS 6 - OpenLDAP - LDAPサーバーの設定 ： Server World](https://www.server-world.info/query?os=CentOS_6&p=ldap)
- [CentOS 6 - OpenLDAP - ユーザーアカウントを追加する ： Server World](https://www.server-world.info/query?os=CentOS_6&p=ldap&f=7)
- [CentOS6 OpenLDAPのインストールと基本設定](http://www.unix-power.net/linux/openldap.html)
- [[ServersMan@VPS] OpenLDAPとPostfixでSMTPサーバーを構築する（その１） | プロジェクト2501](http://www.prj2501.red/wordpress/?p=841)