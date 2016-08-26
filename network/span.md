# SPAN
## 概要
スイッチ上に流れるパケットをキャプチャする方法としてリピータハブを使用する方法とミラーリング機能を使用する方法がある．本稿ではミラーリング機能を使用して行うパケットキャプチャについて説明を行う．

ミラーリング機能とはスイッチの特定ポート上で送受信するパケットを別のポートにコピーする機能である．Cisco Catalystスイッチにおいてこのミラーリング機能はSPAN（Switch Port Analyzer)と呼ばれる．SPANではキャプチャー対象として複数のポートを指定してパケットをキャプチャしたり，VLANべースでキャプチャすることが可能である．

設定を間違えない限りSPANが既存の通信に影響をあたえることはない．とはいえ，宛先ポート（wireshark等パケットキャプチャソフトを起動したコンピュータを接続するポート）の指定先を間違えた場合は障害を引き起こす可能性がある．この宛先ポートはSPAN専用にする必要がある．SPAN専用ポートではSPANセッション以外のトラフィックの送受信はできない．宛先ポートに接続するコンピュータにはIPアドレスを割り振らなくてもパケットキャプチャは可能である．SPANにはローカルSPANとリモートSPANの2種類がある．

## ローカルSPAN
ローカルSPANとは1つのスイッチ内で行うSPANのこと．SPANで指定する送信元ポートと宛先ポートは全て同じスイッチ内にある．
新しくSPANセッションを作成するためには送信元ポート/VLAN（モニターされる側），および宛先ポート（モニターする側）を指定する必要がある．

既存のSPANセッションの削除  
`(config)# no monitor session`

送信元ポート/送信元VLANの設定  
`(config)# monitor session number source [interface id | vlan id] [, | -] [both | rx | tx]`

|monitor sessions|パラメータ説明|
|:---|:---|
|number|SPANセッション番号 1~66まで指定可能|
|interface id|モニターする対象となるポート番号|
|vlan id|モニターする対象となるVLAN番号（interfaceを指定した場合はvlanを指定しない）|
|,|モニターする対象が複数ある場合は[,]で区切る|
|-|モニターする対象が複数ある場合は[-]でレンジを指定する|
|both|送受信トラフィックをモニターする（デフォルト）|
|rx|受信トラフィックのみをモニターする|
|tx|送信トラフィックのみをモニターする|

宛先ポート  
`(config)# monitor session number destination [interface id] [, | -] [encapsulation replicate]`

|monitor sessions|パラメータ説明|
|:---|:---|
|number|SPANセッション番号 sourceで指定したものと同じ値を指定する|
|interface id|ネットワークアナライザが接続するポート番号|
|,|モニターする対象が複数ある場合は[,]で区切る|
|-|モニターする対象が複数ある場合は[-]でレンジを指定する|
|encapsulation replicate|タグ付きのトラフィックを受信したい場合に指定する これを指定しない場合デフォルトでタグ無し時の状態でパケットが宛先ポートに転送されることになる|

### 具体例
F0/10で送受信されるトラフィックをF0/24に接続されたネットワークアナライザで受信する設定  
`(config)# monitor session 1 source interface FastEthernet 0/10`  
`(config)# monitor session 1 destination interface FastEthernet 0/24`

## リモートSPAN
リモートSPAN（RSPAN）は，異なるスイッチ上で行うSPANのこと．RSPANで指定する送信元ポート，送信元VLANは，宛先ポートはNW情の複数のスイッチにまたがることが可能である．なお，RSPANではRSPAN VLANを存在させる必要があり，RSPANを行う全てのスイッチ上でRSPAN VLANを作成する必要がある．

### 具体例
SW1のF0/10で送受信されるトラフィックをSW2のF0/24に接続されたネットワークアナライザで受信する設定
RSPAN用vlanとしてvlan999を設定しトランクポートでスイッチ同士を接続している
```
SW1(config)# vlan 999
SW1(config-vlan)# remote-span

SW1(config)# monitor session 1 source interface FastEthenet 0/10
SW1(config)# monitor sessions 1 destination remote vlan 999

SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
```
```
SW2(config)# vlan 999
SW2(config-vlan)# remote-span

SW2(config)# monitor session 1 source remote vlan 999
SW2(config)# monitor session 2 destination interface FastEthernet 0/24

SW2(config)# interface GigabitEthernet0/1
SW2(config-if)# switchport trunk encapsulation dot1q
SW2(config-if)# switchport mode trunk
```

# 組み込みパケットキャプチャ（EPC）
## 概要
IOS12.4以降で使用可能な機能で，Ciscoルータ上でパケットをキャプチャする機能をEPC（Embedded Packet Capture）という．

キャプチャバッファの作成  
`# monitor capture buffer [buffer-name] size [buffer-size] max-size [element-seize]`  
`[buffer-name]`には保存するバッファ名を指定  
`[buffer-size]`にはバッファのサイズを指定  
`[element-size]`にはパケットごとの最大サイズを指定（オプション）  

キャプチャポイントの作成  
`# monitor capture poiint [ip | ipv6] [cef capture-point-name interface-name interface-type [both | in | out] | process-switched capture-point-name [both |from-us | in | out]]`  
分かりにくい...  
fastEthernet 0/0の送受信パケットをキャプチャする設定する場合は    
`# monitor capture point ip cef cef_point FastEthernet 0/0 both`

キャプチャバッファとキャプチャポイントの関連付け  
`# monitor capture point associate [キャプチャポイント名] [キャプチャバッファ名]`

パケットキャプチャの開始  
`# monitor capture point start all`

パケットキャプチャの停止  
`# monitor capture point stop all`

キャプチャ状況の確認  
`# show monitor capture buffer [キャプチャバッファ名] parameters`

TFTPを用いたエクスポート  
`# monitor capture buffer [キャプチャバッファ名] export [エクスポート先IPアドレス/ファイル名]`

キャプチャバッファ，キャプチャポイントのクリア  
`# monitor capture buffer [キャプチャバッファ名] clear`  
`# monitor capture point disassociate [キャプチャポイント名]`  
`# no monitor capture buffer [キャプチャバッファ名]`  
`# no [キャプチャポイントの作成で入力したコマンド]`  
