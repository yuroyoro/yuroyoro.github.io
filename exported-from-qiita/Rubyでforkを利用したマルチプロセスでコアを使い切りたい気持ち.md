---
title: Rubyでforkを利用したマルチプロセスでコアを使い切りたい気持ち
tags: Ruby
author: yuroyoro
slide: false
---
Rubyで書かれたちょっと重たいバッチ処理があって速くする必要があったので、fork(2)とpipe(2)を使ったマルチプロセス化でコアを活かした並列処理に書き直した話します。

以下の記事に詳しく書いてあるので、TL;DRはそちらを見てな?

[Forking and IPC in Ruby, Part I](http://www.sitepoint.com/forking-ipc-ruby-part/)
[Forking and IPC in Ruby, Part II](http://www.sitepoint.com/forking-ipc-ruby-part-ii/)


[なるほどUnixプロセス ― Rubyで学ぶUnixの基礎 - 達人出版会](http://tatsu-zine.com/books/naruhounix)

# Threadじゃいかんの? — GILについて

並行プログラミングとしてまず最初に思いつくのはマルチスレッド化ですが、RubyにおいてはGVL(Giant VM lock)があるためマルチコアを活かすことは難しいのです。

> ネイティブスレッドを用いて実装されていますが、 現在の実装では Ruby VM は Giant VM lock (GVL) を有しており、同時に実行される ネイティブスレッドは常にひとつです。
> ただし、IO 関連のブロックする可能性があるシステムコールを行う場合には GVL を解放します。その場合にはスレッドは同時に実行され得ます。
[スレッド (Ruby 2.1.0)](http://docs.ruby-lang.org/ja/2.1.0/doc/spec=2fthread.html) より

つまりブロッキング中以外はひとつのThreadしか動けないということです。マルチコア環境ではコア数だけThreadが動いて欲しいけどRubyではそうは行かないのです。
GVLについては、スレッドセーフでないC拡張を考慮してのことらしい (Rubiniusなんかはそのあたり「大丈夫じゃね?」って割り切ってGVLしない模様)。

[CRubyのGVLとビジーループ - kyabの日記](http://d.hatena.ne.jp/kyab/20140215/1392485665)

# forkについて

ではRubyでマルチコアを活用するためにはどうすればよいかというと、fork(2)で複数のプロセスで処理を分散させればよいのです。

> 子プロセスは親プロセスで使われているすべてのメモリのコピーを引き継ぐ。親プロ セスが開いているファイルディスクリプタも同様に引き継ぐ。
[なるほどUnixプロセス ― Rubyで学ぶUnixの基礎 - 達人出版会](http://tatsu-zine.com/books/naruhounix)より

つまり4コアあったら4回forkして処理を分散させればマルチコアを活かして高速化できるわけです。ただしforkすることによるつらみも発生します。

- 親プロセスと同じ量のメモリを消費する
- 子プロセスの管理が煩雑
- 親子間で通信するのに一手間必要(pipeやsocketなど)
- fork自体に多少のオーバーヘッドがある

ひとことで「forkして複数プロセスで動かす」といっても、考えることは色々とあるわけです。
今回の実装では、だいたいこんな感じでやってます。

- 子プロセスとの通信
  - 親から子と、子から親への二つのIO.pipe(pipe(2))を用意する(socketpair(2)を使う手もある)
  - RubyオブジェクトをMarshalでシリアライズしてやりとりする
- 処理の同期/待ち合わせ
  - 子から親のpipeをreadして、blockingすることを利用して待ち合わせる
  - 親側で1プロセスに一つの待ち受け用Threadを起こしてThread#joinで待機
  - Process.waitpid(waitpid(2)) で子プロセスの終了を待つ
- エラー処理
  - 子プロセスで発生したエラーはシリアライズして返す
  - 子プロセスがなんらかの理由で死んだ場合はSIGCHLDをハンドルするか、pipeの切断で検知する
- 親プロセス終了時の子プロセスの回収
  - Kernel#at_exitで子プロセスを回収する
  - 親のSIGINT, SIGQUITは子プロセスにも転送する

# 実装

親からforkして子プロセスで並列処理をさせる手順は、だいたいこんな感じです

- 親側で通信用のIO.pipeオブジェクトを生成しておく
  - forkすると、pipeのファイルディスクリプタを子プロセスでも引き継ぐので、親子間で通信ができる
- 子プロセスをforkする
  - forkした後にpipeの使わない方はcloseしておく
  - 子では、親からpipeに書き込みがあるまでblockingして待つ
- 親では、pipeに書き込むことで子プロセスに処理を依頼する
  - 子プロセスで処理開始
- 親は、子の処理が終わるのを待つ
  - 親はTreadを起こして、子プロセスがpipeに書き込むのを待つ
- 処理が終わったら、子プロセスを終了させてwaitpidで回収する

[parallel.gem](https://github.com/grosser/parallel) でも似たような実装になっています。


図にするとこんなイメージ

![Flowchart (5).png](https://yuroyoro.github.io/exported-from-qiita/images/3e46b6d8-ae5f-dea8-f171-f4588e37cde5.png)


コードはこんなふうになります(いろいろさぼってます)。

```ruby
# 子プロセスを管理するWorkerクラス
class Worker
  attr_reader :pid

  def initialize(&block)
    @child_read, @parent_write = create_pipe # 親から子へのpipe
    @parent_read, @child_write = create_pipe # 子から親へのpipe
    @block = block # forkして実行する処理
  end

  def create_pipe
    # Marshal.dumpの結果はASCII-8BITなのでpipeのエンコーディングもあわせる
    IO.pipe.map{|pipe| pipe.tap{|_| _.set_encoding("ASCII-8BIT", "ASCII-8BIT") } }
  end

  # 子プロセスの起動処理
  def run

    @pid = fork do # forkする

      # 子で使わないpipeは閉じる
      @parent_read.close
      @parent_write.close

      # 親プロセスに起動終了を伝える
      write_to_parent(:ready)

      loop do
        # 親からの依頼待ち
        args = read_from_parent

        # stopが飛んで来たらloopを抜けて子プロセスを終了させる
        break if args == :stop

        # 処理を実行する
        result = @block.call(*args)

        # 結果をpipeに書き込んで完了を親に伝える
        write_object(result, @child_write)
      end

      @child_read.close
      @child_write.close
    end

    wait_after_fork if @pid
  end

  # 子プロセスに処理を行わせる
  def execute(*msg)
    write_to_child(msg)

    Thread.new { read_from_child } # Threadを起こして子からpipeに書き込まれるのを待つ
  end

  def stop
    return unless alive?

    # 子を終了させる
    write_to_child(:stop)

    # waitpidで子プロセスを回収する
    Process.wait(@pid)
  end

  def alive?
    Process.kill(0, @pid)
    true
  rescue Errno::ESRCH
    false
  end

  def write_object(obj, write)
    # RubyオブジェクトをMarshalしてpipeに書き込む
    # 改行をデリミタにする
    data = Marshal.dump(obj).gsub("\n", '\n') + "\n"
    write.write data
  end

  def read_object(read)
    # pipeから読み込んだデータをRubyオブジェクトに復元する
    data = read.gets
    Marshal.load(data.chomp.gsub('\n', "\n"))
  end

  def read_from_child
    read_object(@parent_read)
  end

  def write_to_child(obj)
    write_object(obj, @parent_write)
  end

  def read_from_parent
    read_object(@child_read)
  end

  def write_to_parent(obj)
    write_object(obj, @child_write)
  end

  def wait_after_fork
    @child_read.close
    @child_write.close

    install_exit_handler
    install_signal_handler

    # 子から起動完了が通知されるまで待つ
    Thread.new {
      result = read_from_child
      raise "Failed to start worker pid #{ @pid }" unless result == :ready
      result
    }
  end

  def install_exit_handler
    # Kernel#at_exitで子を回収
    at_exit do
      next unless alive?
      begin
        Process.kill("KILL", @pid)
        Process.wait(@pid)
      rescue Errno::ESRCH
        # noop
      rescue => e
        puts "error at_exit: #{ e }"
        raise e
      end
    end
  end

  def install_signal_handler
    # 親のSIGINT, SIGQUITは子プロセスにも転送する
    [:INT, :QUIT].each do |signal|
      old_handler = Signal.trap(signal) {
        Process.kill(signal, @pid)
        Process.wait(@pid)
        old_handler.call
      }
    end
  end
end
```


このWorkerクラスはこんな風に使います

```ruby
# 4プロセス分Worker作成
workers = 4.times.map{
  Worker.new{|*args| do_something(*args) }
}

# Workerプロセス起動
workers.map(&:run).each(&:join)

# 処理開始
threads = workers.map{|worker|
  worker.execute(*args)
}

# Workerが終わるまで待ち合わせ
threads.each(&:join)

# Workerプロセスを停止
workers.each(&:stop)
```

# サンプルで計測してみる

サンプルとして、青空文庫からテキスト形式で100本ダウンロード、zip解凍し、`OpenSSL::Cipher` でAES256bitで暗号化する処理を、Threadと上記のWorkerクラスでのマルチプロセス化とで速度をtimeコマンドで計測してみたのです(8コア Macbook Pro Late 2013 2.3 GHz Intel Core i7)。

コードは [こちら](https://gist.github.com/yuroyoro/8b060bfaf092395eca07) 。

|                | user       | system       | cpu      | total       |
|:---------------|-----------:|-------------:|---------:|------------:|
| 直列に実行     | 1.68s user | 0.74s system | 40% cpu  | 6.009 total |
| Thread         | 1.95s user | 0.79s system | 279% cpu | 0.979 total |
| マルチプロセス | 3.38s user | 0.97s system | 322% cpu | 1.345 total |

直列に実行して6秒だったのが、Thread化で1秒を切るくらいまで高速化していますね。cpu使用率も279%でコアを活用できていることが分かります。一方、マルチプロセス化は高速化しているがThreadよりは遅い。これは、ダウンロードする処理がボトルネックになっており、IOバウンドな処理だからと推測されます。

そこで、AES暗号化を1000回繰り返すようにしてCPUバウンドな処理にしてどの程度効果があるか試して見ます。結果は以下のようになりました。

|                | user         | system        | cpu      | total         |
|:---------------|-------------:|--------------:|---------:|--------------:|
| 直列に実行     | 133.38s user | 13.01s system | 96% cpu  | 2:31.49 total |
| Thread         | 134.94s user | 14.34s system | 101% cpu | 2:27.46 total |
| マルチプロセス | 180.08s user | 16.32s system | 639% cpu | 30.720 total  |

Threadでは直列に実行した場合とほとんど時間がかわらず、cpu死霊率もあがりません。いっぽうマルチプロセスは、cpu使用率も上がって7倍ほど高速になっています。

IOバウンドな処理ではThreadで充分、CPUバウンドならマルチプロセスにすることで高速化が見込めるということが数値にも表れていますね?

# まとめ

- IOバウンドはThreadで充分
- CPUバウンドならforkしてマルチプロセスが効果あり
- forkするとメモリ使用量や子プロセスの管理などでつらみがある
- [なるほどUnixプロセス ― Rubyで学ぶUnixの基礎 - 達人出版会](http://tatsu-zine.com/books/naruhounix)を読みましょう


# 参考: ActiveRecordを使うには

forkする前に `ActiveRecord::Base.clear_all_connections!` でコネクションを一度リセットして、fork後に `ActiveRecord::Base.establish_connection` で接続しなおす必要があります。

```ruby

# fork前に一度コネクションを切る
ActiveRecord::Base.clear_all_connections!

fork do
  # fork後に子で再接続
  ActiveRecord::Base.establish_connection

  # ...
end

```

# おまけ

[forkを利用したおもしろいプログラム](https://ja.wikipedia.org/wiki/Fork%E7%88%86%E5%BC%BE) です


```
:(){ :|:& };:
```

