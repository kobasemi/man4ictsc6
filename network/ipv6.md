# IPv6
参考URL：[CiscoルータのIPv6基本設定](http://www.unix-power.net/routing/ipv6_settei.html)

## 有効化
`(config)# ipv6 unicast-routing`

## IPv6アドレス（グローバルユニキャスト）の設定
`(config-if)# ipv6 address address/length [ eui-64 ] [ anycast ]`  
オプションである`[eui-64]`をつけると，EUI-64の生成による割り当てを行う．エニーキャストアドレスとして設定したい場合，`[aycast]`をつける．

## IPv6アドレス（リンクローカル）の設定
リンクローカルアドレスは，グローバルユニキャストアドレスを設定することで自動的に生成されるので明示的に設定する必要はない  
`(config-if)# ipv6 address [address] link-local`  

Ciscoルータのインタフェース上でグローバルユニキャストアドレスが設定されていない場合でも，以下のコマンドを設定することでEUI-64形式で自動生成される  
`(config-if)# ipv6 enable`

## IPv6アドレスの自動取得
ルータのインタフェースでIPv6アドレスを自動取得できるようにする設定．オプション設定であり，一般的に設定するコンフィグではない．  

`(config-if)# ipv6 address autoconfig [default]`  

`[default]`を指定することで受信したRA情報に基づきルーティングテーブルにデフォルトルートが登録される．defaultコマンドは機器に1つのインタフェースでのみ設定可  

## IPv6設定情報の表示
`Router# show ipv6 interface [brief]`  

インタフェースに設定されたグローバルユニキャストアドレスと，リンクローカルアドレス，参加しているIPv6のマルチキャストグループ等の情報を確認することが出来る．`[brief]`オプションを付けることで簡易的な情報のみを閲覧することができる

## ルーティングテーブルの表示
`# show ipv6 route`

## 各インタフェースのIPv6アドレス表示
`# show ipv6 int brief`