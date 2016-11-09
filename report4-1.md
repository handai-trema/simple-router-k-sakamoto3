# 課題4-1：ルータのCLIを作ろう
* 学籍番号：33E16009
* 氏名：坂本昂輝

## 課題内容
ルータのコマンドラインインタフェース (CLI) を作成せよ。

次の操作ができるコマンドを作成せよ。

* ルーティングテーブルの表示
* ルーティングテーブルエントリの追加と削除
* ルータのインタフェース一覧の表示
* そのほか、あると便利な機能

コントローラを操作するコマンドの作成方法は、第3回パッチパネルで作った patch_panel コマンドを参考にすること。

## 解答
以下では、各コマンドの説明とそのコマンドの動作確認を行っていく。

### ルーティングテーブルの表示
以下のように、CLI用のRubyスクリプトsimple\_routerを作成した。

* [simple-router-k-sakamoto3/bin/simple\_router]()

ここに、ルーティングテーブルの表示コマンドを実装する。

* ルーティングテーブル表示コマンド

> ./bin/simple\_router display

以下の部分がルーティングテーブルの表示に関わっている。

```
  include Pio

  desc 'Display a routing table'
  arg_name 'routing_table'
  command :display do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      print("destination\tnetmask_length\tnext_hop\n")
      routing_table = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.get_routing_table()
      routing_table.each do |each_netmask_length|
        each_netmask_length.each_key do |each_prefix|
          print IPv4Address.new(each_prefix).to_s, "\t", routing_table.index(each_netmask_length), "\t", each_netmask_length[each_prefix].to_s, "\n"
        end
      end
    end
  end
```
留意すべき行を説明する。

> include Pio

PioライブラリはIPv4Addressを用いるために必要である。

> routing\_table = Trema.trema\_process('SimpleRouter',options[:socket_dir]).controller.get\_routing\_table()

ここでは、./lib/simple_router.rbに記述したget\_routing\_table()メソッドを呼び出す。返り値として、routing\_table.rbで定義された連想配列@dbを返す。

> routing\_table.each do |each\_netmask\_length|
>   each\_netmask\_length.each\_key do |each\_prefix|
>    print IPv4Address.new(each\_prefix).to\_s, "\t", routing\_table.index(each\_netmask\_length), "\t", each\_netmask\_length[each\_prefix].to\_s, "\n"
>   end
> end

routing\_tableに記憶された各項目(destination, netmask\_length, next\_hop)を出力する。

* 動作確認

simple\_router.confに記載された初期状態を表示する。

```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router display
destination	netmask_length	next_hop
0.0.0.0		0				192.168.1.2
```



### ルーティングテーブルエントリの追加と削除

* ルーティングテーブルエントリ追加コマンド

> ./bin/simple\_router add [destination] [netmask\_length] [next\_hop]

* ルーティングテーブルエントリ削除コマンド 

> ./bin/simple\_router delete [destination] [netmask\_length]

simple\_routerに以下のスクリプトを追加する。1つ目の定義がルーティングテーブルエントリの追加、2つ目の定義がルーティングテーブルエントリの削除を表す。

```
  desc 'Add a routing entry'
  arg_name 'destination netmask_length next_hop'
  command :add do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      destination = IPv4Address.new(args[0])
      netmask_length = args[1].to_i
      next_hop = IPv4Address.new(args[2])
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.add_routing_entry(destination, netmask_length, next_hop)
    end
  end
  
  desc 'Delete a routing entry'
  arg_name 'destination netmask_length'
  command :delete do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      destination = IPv4Address.new(args[0])
      netmask_length = args[1].to_i
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.delete_routing_entry(destination, netmask_length)
    end
  end
```

ここでは、各コマンドを定義し、さらに引数を適切に変換して./lib/simple\_router.rbにあるメソッドに渡しているのみである。simple\_router.rbの変更点を以下に記す。

```
  def add_routing_entry(destination, netmask_length, next_hop)
    options = {:destination => destination, :netmask_length => netmask_length, :next_hop => next_hop}
    @routing_table.add(options)
  end

  def delete_routing_entry(destination, netmask_length)
    options = {:destination => destination, :netmask_length => netmask_length}
    @routing_table.delete(options)
  end
```

それぞれ、optionsという連想配列に整形してrouting\_table.rbで定義されたaddメソッド、deleteメソッドに渡すことにより、それぞれの目的が達成される。routing\_table.rbの中身を以下に記載する。

```
  def add(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length][prefix.to_i] = IPv4Address.new(options.fetch(:next_hop))
  end

  def delete(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length].delete(prefix.to_i){|key|
      print("#{key} does not exit.\n")
    }
  end
```

addメソッドは元々実装されていたが、deleteメソッドはそれを参考に新たに作成した。deleteメソッドでは、指定したプレフィックスにマッチする要素をルーティングテーブル@dbから削除する。


* 動作確認

以降では、次の順序で動作確認を行う。

1. ルーティングテーブルの初期状態の表示
2. ルーティングテーブルエントリの追加
3. ルーティングテーブルの表示
4. ルーティングテーブルエントリの削除
5. ルーティングテーブルの表示


```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router display
destination	netmask_length	next_hop
0.0.0.0		0				192.168.1.2

ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router add 192.168.1.3 24 192.168.1.4

ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router display
destination	netmask_length	next_hop
0.0.0.0		0		192.168.1.2
192.168.1.0	24		192.168.1.4

ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router delete 192.168.1.3 24

ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router display
destination	netmask_length	next_hop
0.0.0.0	0	192.168.1.2
```

動作が確認できた。ただし、ここではプレフィックスが一致すれば削除可能であるので、必ずしもdestinationとnetmask\_lengthが一致しなくても良い。例えば、以下の例が挙げられる。

```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router delete 192.168.1.5 24
ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router display
destination	netmask_length	next_hop
0.0.0.0	0	192.168.1.2
```

### ルータのインタフェース一覧の表示

* ルータのインタフェース一覧の表示コマンド

> ./bin/simple\_router interface

simple\_routerにおけるコマンド実装部分は以下となる。

```
  desc 'Display interfaces'
  arg_name 'interface'
  command :interface do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      print("port\tmac_address\t\tip_address\tnetmask_length\n")
      interface = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.get_interface()
      interface.each do |each_interface|
        print each_interface.fetch(:port), "\t", each_interface.fetch(:mac_address), "\t", each_interface.fetch(:ip_address), "\t", each_interface.fetch(:netmask_length), "\n"
      end
    end
  end
```

> interface = Trema.trema\_process('SimpleRouter', options[:socket_dir]).controller.get\_interface()

ここで、simple\_router.rbのget\_interface()メソッドを呼び出している。返り値は、各インターフェースの情報の連想配列を要素にもつ配列である。get\_interface()メソッドは以下となる。

```
  def get_interface()
    interface_array = Array.new()
    Interface.all.each do |each|
      interface_array << {:port => each.port_number, :mac_address => each.mac_address, :ip_address => each.ip_address.value, :netmask_length => each.netmask_length}
    end
    return interface_array
  end
```

この部分は田中くんのレポートを参考にした。元々はInterface.allを返していたが、./bin/simple\_routerとネームスペースが異なるため、その返り値に対してeachメソッドを用いることができなかった。したがって、./lib/simple\_router.rb内部で各インターフェースの情報を示す連想配列を各要素にもつ配列に整形する。

* 動作確認

以下より、動作が確認できた。

```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-k-sakamoto3$ ./bin/simple_router interface
port	mac_address			ip_address	netmask_length
1		01:01:01:01:01:01	192.168.1.1	24
2		02:02:02:02:02:02	192.168.2.1	24
```

## 参考文献
* [Rubylife](http://www.rubylife.jp/)
* [RubyのIPアドレスを文字列に変換する方法](http://trivia.cocolog-nifty.com/blog/2009/03/rubyipaddrip-4c.html)