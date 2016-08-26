## Firewall

VyOSのFirewallには，以下の4つの機能（アクション）がある．

1. accept  
トラフィックを受け入れる 
2. drop  
トラフィックを破棄する
3. inspect  
トラフィックをIPSで処理する
4. reject  
トラフィックを停止する．その際TCPリセットを返す．

また，以下の3つのトラフィックの方向がある．

1. in  
インターフェースへ流入するトラフィックへ適応される  
ex.) ` set interfaces ethernet eth0 firewall in <rule-name> `
2. out  
インターフェースから流出するトラフィックへ適応される  
ex.) ` set interfaces ethernet eth1 firewall out <rule-name> `
3. local  
ルータ宛のトラフィックへのFirewallルールを適応する  
ex.) ` set interfaces ethernet eth2 firewall local <rule-name> `

{host1}→in→{eth0 VyOS eth1}→out→{host2}  
　　　　　　　　　eth2  
　　　　　　　　　　↑  
　　　　　　　　　local  
　　　　　　　　　　↑  
　　　　　　　　　{host3}

- ルールを作っていく段階  
- ルールを設定する段階  

がある．

### ルールを作る

` set firewall name <rule-name> `  
とすることで，任意の名前のルールを作ることができる．  
このルールの内には，基本的に

- default-action
- rule 1~999

を設定する．  
#### default-action
default-actionは，iptablesで言うところのpolicyにあたり，ルールが設定されたインタフェースの流れに来たトラフィックへの，基本的なアクションを意味する．  
accept,drop,inspect,rejectが設定できる．  
ex.) FW1というルールを作り，default-actionをacceptにし，eth0に流入するトラフィックへ適用する．  
```
set firewall name FW1 default-action accept
commit
set interfaces ethernet eth0 firewall in name FW1
```
#### rule 1~999
rule 1~999は，それぞれの数字ごとにルールを設定し，当てはまるトラフィックへのアクションを設定することができる．  
ex.)FW1というルールを作り，rule 1 に送信先IPアドレスが10.10.10.0/24の範囲のもの，かつ送信先ポートが22番ポートのをdropする設定をする．それ以外のトラッフィックはacceptする．これらのルールをeth0に流入するトラフィックへ適用する．   
```
set firewall name FW1 rule 1 destination address 10.10.10.0/24
set firewall name FW1 rule 1 destination port 22
set firewall name FW1 rule 1 action drop
set firewall name FW1 default-action accept
commit
set interfaces ethernet eth0 firewall in name FW1
```


途中にcommitを挟むのは，commitしておかないと初めてその名前のルールを作る場合はVyOS側でルール名が認識されていないからである．（Tabでのサジェストにでてこないだけで，事前にルール名に合致するfirewallを作っていなくても設定することはできる．）

### ルールを設定する
先ほどから設定例に登場する，  
` set interfaces ethernet <interface-name> firewall <in/out/local> name <firewall-name> `  
が，ルールを設定する段階に当たる．  
**ここで設定をしないと，作ったfirewallのルールが適用されることはないため，注意する**   
ex.)eth4からくるルータ本体へ向かうパケットにルールFW2を適用する．  
` set interfaces ethernet eth4 firewall local FW2 `

### firewallの確認
` show firewall `  
を実行すると，現在動いているfirewallのルールを確認することができる．

また，` show firewall `のオプションとして**statistics**がある．    
このコマンドは，設定モードではなく，ユーザーモードで実行できるため，実行する際は設定モードからexitしておく．  
ex.) ` show firewall name FW1 statistics `  
このオプションでfirewallのルールごとのパケット量・バイト数を見ることができる．

さらに，` clear firewall `コマンドの` counters `オプションを組み合わせて使うことで，statisticsで蓄積されているパケット量のカウンタをリセットすることができる．  
ex.) ` clear firewall name FW1 counters `  
firewallのトラブルシュートを行う際は，カウンタを確認しながら行う．



説明文引用：[Vyatta入門(ISBN 978-4-7741-4711-6 C3055)](http://gihyo.jp/book/2011/978-4-7741-4711-6)
