# Router

## ループバックインターフェース
`(config)# interface loopback 0`  
`(config-if)# ip address [IPアドレス] [サブネットマスク]`

ループバックインターフェースは`no shutdown`は必要ない

## バナー
`(config)# banner motd #`  
`[バナーに表示させたいメッセージを入力]`  
`# `

## スタティックルーティング
`(config)# ip route [宛先ネットワークのアドレス] [宛先ネットワークのサブネットマスク] [ネクストホップのアドレス | 自身の送信インターフェース] [AD値] [permanent]`

`[AD値]`と`[permanent] `はオプション

#### デフォルトルートの設定
`(config)# ip route 0.0.0.0 0.0.0.0 [IPアドレス | 自身の送信インターフェース]`

## RIPv2
`(config)# router rip`  
`(config-router)# network [RIPを有効にするインターフェースが属するネットワーク]`  
`(config-router)# version 2`

## サブインターフェースの作成
`(config)# interface [interface-type.subinterface-number]`

具体例  
`(config)# interface FastEthernet0/0.1`

#### サブインターフェースで使用するカプセル化とVLAN番号の指定
`(config-subif)# encapsulation {[isl | dot1q] [VLAN番号] | native}`

#### サブインターフェースにIPアドレスを設定
`(config-subif)# ip address [IPアドレス] [サブネットマスク]`

## OSPF
プロセスの起動  
`(config)# router ospf [プロセスID]`  
有効化するインターフェースとエリアの指定  
`(config-router)# network [ネットワークアドレス|インターフェースのIPアドレス] [ワイルドカードマスク] area [有効化したインターフェースに割り当てたいエリアID]`

具体例  
`(config-router)# network 192.168.1.0 0.0.0.255 area 0`

## show系コマンド
起動中のルーティングプロトコルのステータス  
`# show ip protocols`

ルーティングテーブルを表示  
`# show ip route`

物理，論理インターフェースのステータス  
`# show interfaces`

NAT，NAPTテーブル一覧  
`# show ip nat translations`

#### OSPF
OSPFプロセスの全般的な情報確認  
`# show ip ospf`

OSPFの有効なインターフェース，PID,エリア，コスト値一覧  
`# show ip ospf interface brief`

インターフェースごとのOSPFの動作状況  
`# show ip ospf interface [インターフェース]`

ネイバーリストの確認  
ルータIDはオプション
`# show ip ospf negihbor [ルータID]`
