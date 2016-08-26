# vyosの基本操作方法

## login/logout/shutdown
基本的に，vyosはDebianをベースにしてつくられたOSであるため，通常のサーバと同じような操作はDebianをイメージしながらするといいと思う．

### login
ログインは，普通に  
- ユーザ名
- パスワード  

で行うことができる．
vyosでは，rootの代わりにvyosというユーザが作られ，最初はユーザがvyosのみの状態でインストールされる．
デフォルトでのvyosのパスワードはvyosである．インストールしたら真っ先にパスワードを変えておきたい．

### logout
ログアウトは，一般的なサーバ同じく，  
` exit `  
で行うことができる．

### shutdown
シャットダウンは，    
` sudo shutdown -h now `  

## コマンドの補完
コマンドを途中まで打っている状態で，Tabキーまたは?を押すと，コマンドの補完が行われる．  
コマンドの候補が多数ある(deまで打っていて，destinationとdescriptionが候補にあるなど)場合は，その候補を列挙してくれる．  

また，どんな設定ができるか調査する際にもこの機能が使える．  
例えば，  
` set nat `  
まで打って，次に何を打つべきかわからない時にTabまたは?を押すと，natの次の階層として設定できるものとその説明が列挙される．
## 設定ファイルの構造
設定ファイルは，階層のような構造になっている．下記はその一例である．   
([VyOS公式マニュアル](http://wiki.vyos-users.jp/%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%82%AC%E3%82%A4%E3%83%89)から引用) 
```
interfaces{  
    ethernet eth0 { 
    address dhcp    
        hw-id 00:0c:29:44:3b:0f
    }
    loopback lo {
    }
}
service {
    ssh {
        port 22
    }
}
system {
    config-management {
        commit-revisions 20
    }
    console {
        device ttyS0 {
            speed 9600
        }
    }
    login {
        user vyos {
            authentication {
                encrypted-password ****************
            }
            level admin
        }
    }
    ntp {
        server 0.pool.ntp.org {
        }
        server 1.pool.ntp.org {
        }
        server 2.pool.ntp.org {
        }
    }
    package {
        repository community {
            components main
            distribution stable
            url http://packages.vyos.net/vyos
        }
    }
    syslog {
        global {
            facility all {
                level notice
            }
            facility protocols {
                level debug
            }
        }
    }
}
```
設定ファイルは，この階層構造のどこに設定するかを指定して追加したり削除したりする．  
例えば，上記の設定ファイルに新たにインタフェースeth1を，IPアドレス192.168.0.1で追加する際は，setコマンド(後述)で以下のように階層を順に書いて実行する．  
` set interfaces ethernet eth1 address '192.168.0.1' `

## 設定ファイルの確認・編集
設定ファイルの編集は，コンソール左側のマークが  
` # 　　`  
になっている時のみできる．
この状態のことを設定モードという．

### 設定ファイルの確認
設定ファイルの全体を見るには，showコマンドを使う．  
` $ show `  
これは，設定モードに入っていてもいなくても実行できる．

ちなみに，設定ファイルをsetコマンドの形式で画面に表示するには  
` $ show configration commands `  
を実行すると良い．

### 設定モードに入る
設定モードに入る際は，configureコマンドを使う．  
` $ configure `  
また，設定モードから出る場合は，exitコマンドを使う．  
` # exit `  

### 設定ファイルの編集
設定ファイルの編集には，
1. set
2. delete
3. edit
4. top
5. commit
6. save  
7. compare  
8. rollback  

のコマンドを用いる．これらのコマンドを実行する際は，設定モードに入っておく．

#### 1.setコマンド
設定を追加するときに使うコマンド．追加できると，showコマンドを使って現在の設定ファイルを見たときに，+マークが先頭につく．  追加ではなく変更をすると，>マークが先頭につく．
(設定例)
```
#インタフェースeth3のIPアドレスを10.1.106.1，ネットマスクを/16の形式でかける．
set interfaces ethernet eth3 address ’10.1.106.1/16’
#Source NATの設定で， rule 10に当てはまるもののSNATでの変換後のIPアドレスをマスカレードする
set nat source rule 10 translation address ‘masquerade’
```
#### 2.deleteコマンド
設定を削除するコマンド．削除できたら，showコマンドを使って現在の設定ファイルを見たときに，-マークが先頭につく．
(設定例)
```
#インタフェースeth3のIPアドレスを10.1.106.1，ネットマスクを/16の形式でかけたものを削除する．
delete interfaces ethernet eth3 address ’10.1.106.1/16’
#Source NATの設定で， rule 10の設定を削除する．
delete nat source rule 10
```
#### 3.editコマンド
設定モードに入っている際に，設定ファイル内の階層を移動するコマンド．  
例えば，setコマンドだけだとインタフェースの設定をする際に下記のようなコマンドを打つ．
```
set interfaces ethernet eth3 description ‘internet’
set interfaces ethernet eth3 address ’10.1.106.1/16’
set interfaces ethernet eth4 description ‘router_router’
set interfaces ethernet eth4 address ’192.168.0.254/24’
set interfaces ethernet eth5 description ‘dmz’
set interfaces ethernet eth5 address ’192.168.1.254/24’
set interfaces ethernet eth6 description ‘steptodmz’
set interfaces ethernet eth6 address ’192.168.100.254/24’
```
しかし，editコマンドを使うとこのようになる．
```
edit interfaces ethernet
set eth3 description ‘internet’
set eth3 address ’10.1.106.1/16’
set eth4 description ‘router_router’
set eth4 address ’192.168.0.254/24’
set eth5 description ‘dmz’
set eth5 address ’192.168.1.254/24’
set eth6 description ‘steptodmz’
set eth6 address ’192.168.100.254/24’
top
```
このように，途中まで同じ階層のコマンドを打つ際に，editコマンドで階層を移動しておくと，省略してコマンドを打つことができる．
#### 4.topコマンド
editコマンドで移動したカレントの階層を，ルートの階層まで戻すコマンド．
#### 5.commitコマンド
追加・削除した設定ファイルを一時保存する．commitコマンドを実行後にshowコマンドで設定ファイルを確認すると，今まで付いていた+や-の記号はなくなる．  
commitコマンドを実行すると設定は有効化されるが，再起動すると設定ファイルの変更点が削除されてしまう．

通常，commitコマンドが実行されていない(設定ファイルが保存されていない)と，設定モードを終了することができない．
commitせずに設定モードを出るなら，  
` # exit discard `  
を使って，設定モードから出る．(設定したものは全て削除される) 
#### 6.saveコマンド
commitコマンドで保存した設定を，本格的に記録する．saveコマンドで変更を保存すると，再起動しても設定が消えなくなる．
#### 7.compareコマンド
vyosは設定ファイルのバックアップをとっているため，compareコマンドで設定を比べることもできる．また，  
` compare 1 `
などとすると，1つ前の設定ファイルと比べてくれる．
#### 8.rollbackコマンド
設定ファイルのバックアップから，過去の設定ファイルを復元してくる．  
` rollback 1 `
とすると，1つ前の設定ファイルに書きかわる．  
コマンド実行後再起動されるため，再起動してはいけないシステムの場合使えない．

#### 基本的な流れ
設定ファイルの編集だが，基本的に以下のような流れになる．  

1. set/delete/edit/top
2. commit
3. save

