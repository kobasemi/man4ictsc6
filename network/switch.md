# Switch

## VLANの作成
`(config)# vlan [VLAN番号]`
`(config-vlan)# name [VLAN名]`

`[VLAN名]`は管理用の名前（オプション）

## VLANの割り当て
`(config-if)# switchport mode [access | trunk]`
`(config-if)# switchport [access | trunk] allowe vlan [VLAN番号]`

## トランキング
`(config-if)# switchport mode trunk`
`(config-if)# switchport trunk encapsulation [isl | dot1q | negotiate]`
`(config-if)# switchport trunk allowed vlan [vlan-id]`
`(config-if)# switchport mode [dynamic {auto | desirable} | trunk]`

#### 既存のトランクに新しいvlanを追加
`(config-if)# switchport trunk allowed vlan add [vlan-id]`

## RSTP
`(config)# spanning-tree mode rapid-pvst`

無効化する場合
`(cnfig)# no spanning-tree vlan [VLAN番号]`

#### スイッチプライオリティ値の変更
`(config)# spanning-tree vlan [VLAN番号] priority 4096`

#### 任意のスイッチをルートブリッジに設定する
`(config)# spanning-tree vlan [VLAN番号] root primary(secondary) [diameter net-diameter [hello-time seconds]]`
自動的に現在のルートブリッジを検出して目的のスイッチがルートになるようにスイッチのプライオリティ値を下げる．セカンダリルートブリッジとして設定することも可能．
`diameter`は任意の二つのエンドーステーション間の最大スイッチ数を指定
`hello-time`はルートスイッチでconfiguration BPDUが生成される間隔を1~10の範囲で指定．

#### ポートコストの設定
アクセスポートの場合
`(config-if)# spanning-tree cost [コスト値]`
トランクポートの場合
`(config-if)# spanning-tree vlan [VLAN番号] cost [コスト値]`

## PVST
`(config)# spanning-tree mode pvst`

無効化する場合
`(config)# no spanning-tree vlan [VLAN番号]`

## EtherChannel
`(config-if)# switchport mode access`
`(config-if)# switchport access vlan [VLAN番号]`
or
`(config-if)# switchport mode trunk`

`(config-if)# channel-group [チャンネルグループ番号] mode {auto [non-silent] | desirable [non-silent] | on | [active | passive]}`

## show系コマンド
リンクアップ状態，duplex/speedの状態，VLAN番号，メディアタイプ
`# show interfaces status`

インターフェースごとのエラーカウントの一覧情報
`# show interfaces count error`

各VLANインターフェースに割り当てられたIPアドレス一覧
`# show ip interface brief`

EtherChannelのグループ，使用プロトコル，物理ポート
`# show etherchannel summary`

VLANデータベース，ステータス，割り当てポートの確認
`# show vlan`

トランクポート，許可VLAN，ステータス
`# show interfaces trunk`

各VLANごとのスパニングツリーのステータス
`# show spanning-tree`

VLAN設定の確認
`(config)# show vlan`

ポートのモード，トランクのカプセル化，プルーニングに関する情報などの確認
`(config)# show interface [ポート番号] switchport`

## その他
