# Router&Switch共通

## hostname
`(config)# hostname [ホスト名]`

## IPアドレス
`(config)# interface [インタフェース]`  
`(config-if)# ip address [IPアドレス] [サブネットマスク]`  
`(config-if)# no shutdown`  
「スイッチ自体」にIPアドレスを設定する場合`[インターフェイス]`は`vlan 1`となる

## デフォルトゲートウェイ
`(config)# ip default-gateway [IPアドレス]`  
異なるネットワークセグメントからスイッチに管理アクセスするために必要

## Local user and password
`(config)# username [ユーザ名] password [パスワード]`

## Enable secret password
ユーザモードから特権モードへ移行する際必要となるパスワード  
`(config)# enable password [パスワード]`  
`(config)# enable secret [パスワード]`

`password `の場合は平文で，`secret`の場合は暗号化されて設定ファイルに保存される

## コンソール/VTYログイン
ログインパスワードはNW機器に接続し，ログインする時点で必要となるパスワード  
`(config)# line [接続ポート]`  
`(config-line)# login`  
`(config-line)# password [パスワード]`

`[接続ポート]`に入力するのはコンソール接続の場合`console 0`，telnetやsshの場合`vty 0 4`（vtyポート0~4番）  
接続先のNW機器にtelnetのパスワードを設定していないとtelnet接続が許可されない  
特権モードのパスワードを設定していないと，telnet先で特権モードに入れない

## exec-timeout
なにも操作をしなかった際にセッションが切れログアウトされるまでの時間（デフォルトは10分）を変更するコマンド  
`(config)# line [接続ポート]`  
`(config-line)# exec-timeout [minute] [second]`

## Service password encryption
`(config)# service password-encryption`

設定ファイル内のパスワードが全て暗号化される（復号不可）  
このコマンドを入力した場合，以降に入力されるパスワードも全て暗号化される

## copy run start
設定の保存（running-configの保存）  
`# copy running-config startup-config`

省略可能  
`# copy run start`

copyコマンドの文法  
`# copy [コピー元] [コピー先]`

## ping
`> ping [宛先IPアドレス | 宛先ホスト名]`

## SSH
`(config)# line vty 0 4`  
`(config-line)# login local`  
`(config-line)# transport input ssh `

## traceroute
`> traceroute [宛先IPアドレス| 宛先ホスト名]`

## show cdp neighbors
隣接機器の情報を表示  
`# show cdp neighbors`

より詳細な情報（L3の情報）を表示  
IPv6の情報も表示してくれる  
`# show cdp naighbors detail`

## 複数インターフェースをまとめて設定
`(config)# interface range [インターフェイス]`

具体例
`(config)# interface range fastEthernet 0/7 - 24`

## show系コマンド
IPアクセスリストの適用場所，適用方向の確認  
`# show ip interface`

インターフェースの状態やIPアドレスの確認  
`# show interfaces`

ARPテーブルの確認  
`# show arp`

ネイバーテーブル（v6版ARPテーブル）確認  
`# show ipv6 neighbors`
