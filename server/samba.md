# Sambaサーバ構築

OS：Ubuntu 14.04.4

## ソフトウェアのインストール

以下のコマンドを実行して，必要なソフトウェアをインストールする．

```
# apt-get install samba
```

## Samba用共有ディレクトリ作成

Sambaサーバの設定をする前に，Sambaサーバで使用する共有ディレクトリを以下のコマンドで作成する．

```
# mkdir -p /var/samba/public
# chmod 777 /var/samba/public
```

## Sambaサーバ設定

`/etc/samba/smb.conf`を編集し，Sambaサーバを設定する．

```
#========== Global Settings ==========
[global]
unix charset = UTF-8
dos charset = CP932

## Browsing/Identification ##
# WindowsのWORKGROUP/NT-domainに合わせる
workgroup = WORKGROUP

server string = %h server (Samba, Ubuntu)

dns proxy = no

#### Networking ####
# SambaサーバをbindできるIPアドレスを設定する
interfaces = 127.0.0.0/8 192.168.2.0/24

# 上で設定したもののみにbindを許可する
bind interfaces only = yes

#### Debugging/Accounting ####
log file = /var/log/samba/log.%m

max log size = 1000

syslog = 0

panic action = /usr/share/samba/panic-action %d

###### Authentication ######
# Sambaサーバをどのモードで動かすかを指定する
server role = standalone server

# 暗号化されたパスワードを使っている場合，パスワードDBのタイプを指定
passdb backend = tdbsam

obey pam restrictions = yes

# passdbで変更があった場合に，Samba側で同期するか
unix password sync = yes

# Debian GNU/LinuxなシステムでUnixパスワードの同期をする際は，以下が必要
passwd program = /usr/bin/passwd %u
passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .

# SMBクライアントからパスワード変更の要求があった際に，passwd programよりPAMを優先するか
pam password change = yes

map to guest = bad user

########## Misc ##########
usershare allow guests = yes

***省略***

#共有設定
[Public]
  path = /var/samba/public
  writable = yes # 書き込み可能
  guest ok = yes # ゲストユーザOK
  guest only = yes # 全ユーザをゲストとして扱う
  create mask = 0777 # ファイルに対して設定できるアクセス権限の上限値を指定
  directory mask = 0777 # ディレクトリに対して設定できるアクセス権限の上限値を指定
  force create mode = 0777 # 作成したファイルのアクセス権限が必ず0777になる
  force directory mode = 0777 # 作成したディレクトリのアクセス権限が必ず0777になる
  share modes = yes # 複数人が同一ファイルに同時でアクセスした際に警告
```

## Samba起動

以下のコマンドでSambaサーバを起動する．

```
# initctl start smbd
```

なお，デフォルトでシステム起動時に自動起動するようになっている．これは`/etc/init/smbd.conf`の`start on`で制御可能である．

## クライアント：共有ディレクトリのマウント

Arch Linuxで先ほど作成した共有ディレクトリをマウントする．

まず，以下のコマンドでクライアントソフトウェアをインストールする．

```
# pamcan -S smbclient
```

インストールできたら，以下のコマンドでSambaサーバの公開情報を表示してみる．

```
$ smbclient -L samba.ictsc6.local -U%
```

共有ディレクトリをマウントするにはまず，マウント先となるディレクトリを作成する必要がある．

```
# mkdir /mnt/mountpoint
```

ディレクトリ作成後，以下のコマンドで共有ディレクトリをマウントできる．

```
# mount -t cifs //samba.ictsc6.local/Public /mnt/mountpoint -o user=testuser,workgroup=WORKGROUP
```

今回作成した共有ディレクトリは，ゲストユーザでもマウント可能である．つまり，-oオプションでユーザ名を指定せずともマウントできる．-oオプションでパスワードを指定していないので接続後パスワード入力を促されるが，そのままEnterを押せば接続できる．

マウント後，以下のコマンドを実行すれば共有ディレクトリがマウントできているかを確認できる．

```
$ df -h
```

`/mnt/mountpoint/`以下に適当なファイルを作成した後，Sambaサーバ側で`/var/samba/public/`の中に作成したファイルが存在していれば，正常にマウントされている．

アンマウントする際は，以下のコマンドを実行する．

```
# umount /mnt/mountpoint
```

## 参考

- [Ubuntu 14.04 LTS : Sambaサーバー : フルアクセスの共有フォルダ作成 ： Server World](https://www.server-world.info/query?os=Ubuntu_14.04&p=samba)
- [Samba - ArchWiki](https://wiki.archlinuxjp.org/index.php/Samba)
- [共有フォルダの運用パラメーター | Think IT（シンクイット）](https://thinkit.co.jp/article/766/1)