# Comparison of CentOS / Ubuntu versions

ここでは，CentOSの6と7，Ubuntuの14.04.4と16.04のそれぞれについて，サービスの管理方法やログ，設定ファイルのパス，ファイアウォールを管理するプログラム，ディストリビューション・バージョン確認の方法などについてまとめる．

## CentOS 6 and 7

### サービス管理

|         | CentOS6                                  | CentOS7                                  |
| :-----: | ---------------------------------------- | ---------------------------------------- |
| 管理システム  | SysVinit, Upstart                        | Systemd                                  |
|   一覧    | chkconfig —list, initctl list            | systemctl —type service                  |
|   確認    | service hoge status, initctl status hoge | systemctl status hoge                    |
|   起動    | service hoge start, initctl start hoge   | systemctl start hoge                     |
|   停止    | service hoge stop, initctl stop hoge     | systemctl stop hoge                      |
|  リロード   | service hoge reload, initctl reload hoge | systemctl reload hoge                    |
|   再起動   | service hoge restart, initctl restart hoge | systemctl restart hoge                   |
| 自動起動ON  | chkconfig hoge on, /etc/init/以下の設定ファイルにおけるstart onをアンコメント | systemctl enable hoge                    |
| 自動起動OFF | chkconfig hoge off, /etc/init/以下の設定ファイルにおけるstart onをコメントアウト | systemctl disable hoge                   |
| ファイルパス  | /etc/init.d/, /etc/init/                 | /usr/lib/systemd/system/, /etc/systemd/system/ |

### サービスログ

|        | CentOS6                 | CentOS7                                  |
| :----: | ----------------------- | ---------------------------------------- |
| ファイルパス | /var/log/               | /var/log/, /run/log/journal/             |
|  確認方法  | less  /var/log/secure   | less /var/log/secure, journalctl -u sshd |
|   監視   | less +F /var/log/secure | less +F /var/log/secure, journalctl -f -u sshd |

### 設定ファイルパス

|          | CentOS6 | CentOS7 |
| :------: | ------- | ------- |
| 設定ファイルパス | /etc/   | /etc/   |

### ネットワーク設定

|        | CentOS6                                  | CentOS7                                  |
| :----: | ---------------------------------------- | ---------------------------------------- |
| 設定ファイル | /etc/sysconfig/network-scripts/ifcfg-eth0 | /etc/sysconfig/network-scripts/ifcfg-eth0 |
|  コマンド  | system-config-network(-tui)              | nmctl                                    |

### ファイアウォール

|       | CentOS6  | CentOS7             |
| :---: | -------- | ------------------- |
| プログラム | iptables | iptables, firewalld |

### ディストリビューション，バージョン確認

```
$ uname -a
$ cat /etc/*-release
$ cat /proc/version
$ cat /etc/redhat-release
```

## Ubuntu 14.04.4 and 16.04

### サービス管理

|         | Ubuntu14.04.4                            | Ubuntu16.04                              |
| :-----: | ---------------------------------------- | ---------------------------------------- |
| 管理システム  | SysVinit, Upstart                        | SysVinit, Upstart, Systemd               |
|   一覧    | service —status-all, sysv-rc-conf —list, initctl list | service —status-all, sysv-rc-conf —list, systemctl —type service |
|   確認    | service hoge status, initctl status hoge | service hoge status, systemctl status hoge |
|   起動    | service hoge start, initctl start hoge   | service hoge start, systemctl start hoge |
|   停止    | service hoge stop, initctl stop hoge     | service hoge stop, systemctl stop hoge   |
|  リロード   | service hoge reload, initctl reload hoge | service hoge reload, systemctl reload hoge |
|   再起動   | service hoge restart, initctl restart hoge | service hoge restart, systemctl restart hoge |
| 自動起動ON  | sysv-rc-conf hoge on, /etc/init/以下の設定ファイルにおけるstart onをアンコメント | sysv-rc-conf hoge on, /etc/init/以下の設定ファイルにおけるstart onをアンコメント, systemctl enable hoge |
| 自動起動OFF | sysv-rc-conf hoge off, /etc/init/以下の設定ファイルにおけるstart onをコメントアウト | sysv-rc-conf hoge off, /etc/init/以下の設定ファイルにおけるstart onをコメントアウト, systemctl disable hoge |
| ファイルパス  | /etc/init.d/, /etc/init/, (/usr/lib/systemd/), (/etc/systemd/system/) | /etc/init.d/, /etc/init/, /usr/lib/systemd/, /etc/systemd/system/ |

### サービスログ

|        | Ubuntu14.04.4             | Ubuntu16.04                              |
| :----: | ------------------------- | ---------------------------------------- |
| ファイルパス | /var/log/                 | /var/log/, /run/log/journal/             |
|  確認方法  | less /var/log/auth.log    | less /var/log/auth.log, journalctl -u apache2 |
|   監視   | less +F /var/log/auth.log | less +F /var/log/auth.log, journalctl -f -u apache2 |

### 設定ファイルパス

|          | Ubuntu14.04.4 | Ubuntu16.04 |
| :------: | ------------- | ----------- |
| 設定ファイルパス | /etc/         | /etc/       |

### ネットワーク設定

|        | Ubuntu14.04.4           | Ubuntu16.04             |
| :----: | ----------------------- | ----------------------- |
| 設定ファイル | /etc/network/interfaces | /etc/network/interfaces |
|  コマンド  | ifconfig, ip            | ifconfig, ip            |

### ファイアウォール

|       | Ubuntu14.04.4 | Ubuntu16.04 |
| :---: | ------------- | ----------- |
| プログラム | iptables, ufw | iptables    |

### ディストリビューション，バージョン確認

```
$ uname -a
$ cat /etc/*-release
$ lsb_release -a
$ cat /proc/version
$ cat /etc/debian_version
```