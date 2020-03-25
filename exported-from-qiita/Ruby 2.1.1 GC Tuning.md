---
title: Ruby 2.1.1 GC Tuning
tags: Ruby:2.1.1
author: yuroyoro
slide: false
---
Ruby2.1では、RGenGCによりかなりパフォーマンスが改善されている。
また、チューニングパラメータが増えているが、まとまった日本語の解説が無かったので書いてみた。

間違いがある可能性があるので、指摘は歓迎です。


# RGenGCとは

RGenGC(Restricted Generational Garbage Collection)については、まずはこれを読むべし

+ [www.atdot.net/~ko1/activities/rubyconf2013-ko1_pub.pdf](http://www.atdot.net/~ko1/activities/rubyconf2013-ko1_pub.pdf)
+ [www.atdot.net/~ko1/activities/2014_rubyconf_ph_pub.pdf](http://www.atdot.net/~ko1/activities/2014_rubyconf_ph_pub.pdf)
+ [Ruby 2.1: RGenGC · computer talk by @tmm1](http://tmm1.net/ruby21-rgengc/)
+ [RGenGCの発表資料を読んだ - oupoの日記](http://d.hatena.ne.jp/oupo/20130605/1370434919)

上記の資料でほぼ概要はつかめるはず。

##  Heapレイアウト

+ HeapはSlot(とPage)からなる
+ Rubyオブジェクト(`RVALUE`)が生成されるとSlotに配置される
+ 各Slotのサイズは40Byte
+ HeapはPageに分割されて、各Page毎にSlotがある
+ Pageは"Eden"と"Tomb"に分割されている。
+ Edenはオブジェクトが配置されるPage
+ Tombは、Edenが無くなった場合に昇格する予備Page
+ 空きSlotが少なくなるとGCが発生

Heapの構造については、以下の「オブジェクトの管理」あたりを

[第5章 ガ－ベージコレクション](http://i.loveruby.net/ja/rhg/book/gc.html)

`RVALUE`のサイズは5word(40Byte)だが、このサイズを8wordに拡張することで、cpuのcachelineをまたぐことがなくなりcache効率が向上、拡張した領域にいろいろと詰め込むことで速度改善する、というPullRequestがある。パない。

+ [卜部昌平のあまりreblogしないtumblr - ruby/ruby#495日本語解題](http://shyouhei.tumblr.com/post/72765924962/ruby-ruby-495)
+ [[Feature #9362] Minimize cache misshit to gain optimal speed #495](https://github.com/ruby/ruby/pull/495)

## 世代別GC

+ 95%のオブジェクトは最初のGCで回収される
+ Mark & Sweep (マイナーGCもメジャーGCも)
+ Rootから到達可能なオブジェクトをmark
+ Oldオブジェクト : markされたオブジェクト
+ markされてないオブジェクトを 解放、再配置
+ メジャーGCは全オブジェクトをtravrseする (遅い)
+ マイナーGCでは、Oldオブジェクトから先の参照をmarkしない(ココが高速化のポイント)

## WriteBarrier

問題点

+ マイナーGCでは、Oldオブジェクトから先の参照をtraverseしない
+ M&Sで、(old -> new) な参照があると、mark漏れが発生。生きてるオブジェクトをGCしてしまう

対策

+ (old -> new) な参照が生まれると、参照元のOldオブジェクトを覚えておく(Remember Set)
+ Remember Setに含まれるオブジェクトはM&SでRootとして扱う -> mark missを防ぐ
+ オブジェクトへの書き込み時に、(old -> new)な参照かをチェックするのがWriteBarrier

## Normal/Shadyオブジェクト

+ Rubyの世界で発生したオブジェクトは、WriteBarrier保護されている(Normal Object)
+ C拡張から生成されたオブジェクトは、WriteBarrier保護されていない(Shady Object)
+ Shady Objectはoldにならない
+ つまり、マイナーGCのMark時にShady Objectから先の参照もmark対象となる
+ WriteBarrierによりC拡張側のメモリを意図せず解放しないようになっている

# GCはいつ発生する?

マイナーGC

- Heap内の空きSlotが無くなった場合 (`if [# of Total slots] * 0.7 < [# of Living objects]`)
- mallocしたsizeが一定量を超えた場合(後述 -> 「malloc_limit関連」)

メジャーGC

- Oldオブジェクト数が閾値を超えた場合
- RemberSet内のShady Object数が閾値を超えた場合

## 例 : Object.newからのGC発生を深掘りしてみる

`object.c`、`gc.c`あたりを見てみると

1. Object.newを呼び出す
1. [object.c : rb_class_new_instance()関数が呼ばれる](https://github.com/ruby/ruby/blob/v2_1_1/object.c#L1837)
1. [object.c : rb_obj_alloc()関数を呼び出す](https://github.com/ruby/ruby/blob/v2_1_1/object.c#L1841)
1. [object.c : rb_obj_alloc()関数では、allocatorとして定義されている関数を呼び出してobjectをheapに確保する](https://github.com/ruby/ruby/blob/v2_1_1/object.c#L1809)
1. [object.c : allocatorの実体は、rb_define_alloc_func()で登録されたrb_class_allocate_instance()関数で、その実体はNEWOBJ_OFマクロ](https://github.com/ruby/ruby/blob/v2_1_1/object.c#L1820)
1. [include/ruby/ruby.h : NEWOBJ_OFの定義はrb_newobj_of()関数](https://github.com/ruby/ruby/blob/v2_1_1/include/ruby/ruby.h#L694))
1. [gc.c : rb_newobj_of()は単にnewobj_of()を呼び出すだけ ](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1351)
1. [gc.c : newobj_of()内で、heap_get_freeobj()を呼び出している。ここがObjectをHeapにallocateする場所のもよう](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1300)
1. [gc.c : heap_get_freeobj()でheap_get_freeobj_from_next_freepage()を呼び出している](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1256)
1. [gc.c : heap_prepare_freepage()で空きPageが取得できるまでheap_prepare_freepage()を呼ぶ](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1234)
1. [gc.c : heap_increment()がFALSEを返した場合はGCが実行される模様](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1198)
1. [gc.c : heap_increment()は、heap_pages_increment = 0ならFALSEを返す](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1186)
1. [gc.c : heap_pages_incrementは、heap_set_increment()で設定される値。利用できるHeap内のPageの残数っぽい?](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L1173)

このあたりで力尽きた


# GCのパラメータ

GCのパラメータは`gc.c`に定義してあるぽい。環境変数経由で設定する

[ruby/gc.c at v2_1_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L5716)

初期値はこのあたりに

[ruby/gc.c at v2_1_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L100)

## 基本

- `RUBY_GC_HEAP_INIT_SLOTS`
  (default : 10000)

  最初にHeep上にAllocateするSlot数

- `RUBY_GC_HEAP_FREE_SLOTS`
  (default: 4096)

  GC後に最低限確保しておく空きSlot数
  この値を下回ると、追加でSlotを確保しようとする

  なお、総Slot数に対する空きSlot数の割合は、30%から80%以内に収まるように実装されている
  [ruby/gc.c at v2_1_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v2_1_1/gc.c#L2862-L2869)

- `RUBY_GC_HEAP_GROWTH_FACTOR`
  (new from 2.1)
  (default : 1.8)

  Slotを確保する際の増加倍率。
  Slotを増やす際には、現在のSlot数にこの倍率をかけた値まで拡張される

- `RUBY_GC_HEAP_GROWTH_MAX_SLOTS`
  (new from 2.1)
  (default 0)

  一度に確保するSlot数の上限。

  `RUBY_GC_HEAP_GROWTH_FACTOR`が1.0以上に設定されている場合、 HEAPが不足するたびにSlot数が増加するが、このパラメータにより一度に確保するSlot数の上限を設定できる。

  ただし、defultは0に設定されており、0の場合はSlot数の上限は有効にならない。

- `RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR`
  (new from 2.1.1)
  (default : 2.0)

  Oldオブジェクト数が一定数を超えるとメジャーGCが実行される。

  `(oldobject数) > (R * N)`

  R : この値(`RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR`)
  N : 前回メジャーGCが終了した時点でのoldobject数

  初期値では2.0なので、oldobject数が2倍になる毎にメジャーGCされることになる。

## malloc_limit関連

RubyのHeap領域内のSlotには`RVALUE`オブジェクトが配置される。Slotのサイズは40byteで、`RVALUE`構造体それ自体はそのサイズ内に収まるが、Stringオブジェクトの文字列本体などはHeap外にメモリを確保する必要がある。
このメモリの確保は`ruby_xmalloc()`によって行われる。

VMは、`ruby_xmalloc()`によって確保されて未解放の領域サイズ(byte)を管理している。
この値を`malloc_increase`という。

- `RUBY_GC_MALLOC_LIMIT`
  (default: 16MB)

  `malloc_increase`が`RUBY_GC_MALLOC_LIMIT`の値を超えた場合、マイナーGCを実行する(空きSlotの有無にかかわらず)

  defaultは16MBなので、Heap外に16MB確保する毎にマイナーGCが実行されることになる。
  この初期値は古いマシンを基準に設定されているので、ほとんどのRailsアプリケーションではこの値を増加させたほうがよい。

  大きめに確保しておくことで、`ruby_xmalloc()`をトリガーとしたマイナーGCの発生頻度を押させることができる(rssの肥大とトレードオフ)

  `GC.stat`で、`malloc_increase`の値を確認できる

- `RUBY_GC_MALLOC_LIMIT_GROWTH_FACTOR`
  (new from 2.1)
  (default : 1.4)

  2.1からは、`RUBY_GC_MALLOC_LIMIT`の値(`malloc_limit`)は、`malloc_increase`が閾値を超える毎に、この値を倍率として更新される。

  ただし、`RUBY_GC_MALLOC_LIMIT_MAX`を超えて拡張することはない。

- `RUBY_GC_MALLOC_LIMIT_MAX``
  (default: 32MB)

  `RUBY_GC_MALLOC_LIMIT_GROWTH_FACTOR`によって`malloc_limit`は拡張されるが、その最大値。

## old_malloc_limit関連

  Oldオブジェクトが`ruby_xmalloc()`によって確保しているサイズを閾値としたメジャーGCについては、以下のパラメータで調整できる。
  各パラメータの位置付けは、`malloc_increase`を元にしたGCと同じ扱いである

- `RUBY_GC_OLDMALLOC_LIMIT` (default : 16MB)
- `RUBY_GC_OLDMALLOC_LIMIT_MAX` (default :128MB)
- `RUBY_GC_OLDMALLOC_LIMIT_GROWTH_FACTOR` (default : 1.2)

# GCの状況確認 : GC.statについて

GCの実行回数などの統計情報は、`GC.stat`で確認できる

それぞれの値の意味については、以下の資料が詳しい

+ [Watching and Understanding the Ruby 2.1 Garbage Collector at Work - Thorsten Ball](http://thorstenball.com/blog/2014/03/12/watching-understanding-ruby-2.1-garbage-collector/)

が、簡単に解説してみる

```ruby
GC.stat

{
 :count                         => 162,            # GCの実行回数(minor + fullの合計)
 :minor_gc_count                => 137,            # マイナーGC実行回数
 :major_gc_count                => 25,             # メジャーGC実行回数
 :total_allocated_object        => 210784578,      # Heapに確保された通算のオブジェクト数
 :total_freed_object            => 199431871,      # Heapから回収された通算のオブジェクト数
 :heap_length                   => 75618,          # Heap用に確保されたPage数
 :heap_used                     => 43315,          # 実際に使用されているPage数(Eden + Tomb)
 :heap_eden_page_length         => 42427,          # "Eden" Page数
 :heap_tomb_page_length         => 888,            # "Tomb" Page数
 :heap_live_slot                => 11352707,       # 使用Slot数
 :heap_free_slot                => 6302456,        # 空きSlot数
 :heap_final_slot               => 0,              # finalizerを実行すべきSlot数
 :heap_swept_slot               => 6297729,        # SweepされたSlot数(各Pageをsweepするたびにresetされる)
 :heap_increment                => 32303,          # 未使用Page数 (:heap_used + :heap_increment = :heap_length)
 :remembered_shady_object       => 39172,          # Remeber Set内のshady object数(メジャーGC毎にresetされる)
 :remembered_shady_object_limit => 43012,          # remembered_shady_objectがこの値を超えるとメジャーGCが発生する. メジャーGC毎にRUBY_GC_OLDMALLOC_LIMIT_GROWTH_FACTORを倍率として更新される
 :old_object                    => 10920042,       # Oldオブジェクト数
 :old_object_limit              => 20146314,       # old_objectがこの値を超えるとメジャーGCが発生する. メジャーGC毎にRUBY_GC_OLDMALLOC_LIMIT_GROWTH_FACTORを倍率として更新される
 :malloc_increase               => 40715312,       # `ruby_xmalloc()`によって確保されて未解放の領域サイズ(byte)
 :malloc_limit                  => 50000000,       # malloc_increaseがこの値を超えるとマイナーGCが発生するj
 :oldmalloc_increase            => 83253560,       # Oldオブジェクトが`ruby_xmalloc()`で確保している領域サイズ(byte)
 :oldmalloc_limit               => 134217728       # oldmalloc_increaseがこの値を超えると、メジャーGCが発生する
}
```
gc_tracer.gemを利用すると、`GC.stat()`や`GC.latest_gc_info()`の結果をGCの開始、mark&sweeepの開始/終了のタイミングでloggingできる模様。
[ko1/gc_tracer](https://github.com/ko1/gc_tracer)

# チューニング戦略

GCのチューニングはアプリケーションの特性にもよるが、メモリ使用量とスループットのトレードオフと考えてよい。

まず、`malloc_limit`関連で、`RUBY_GC_MALLOC_LIMIT`は引き上げておいた方がよいだろう。
どの程度に設定するかはプロセスのrssの増加量にもよるが、手元のアプリケーション(Rails Unicorn)では64MBから128MB程度の値で、速度が安定した。

Slot数についてだが、基本的には、初期確保するSlot数(`RUBY_GC_HEAP_INIT_SLOTS`)を増やすことで、 立ち上がりからGCが暖まるまでの時間を短縮できる。

GC後の空きSlot数(`RUBY_GC_HEAP_FREE_SLOTS`)を多くしておくことで、GC後に使えるSlotが増えて、次回GC実行までのインターバルを長くし、トータルのGC実行時間を短くできる。
または、`RUBY_GC_HEAP_GROWTH_FACTOR`の値を大きくすることで、Heapの拡大率を上げるという方法もある。

無論、これらの値を多くすることはメモリ使用量とのトレードオフである。

また、`RUBY_GC_HEAP_FREE_SLOTS`を多くしておくと、`RUBY_GC_HEAP_GROWTH_FACTOR`の倍率だけSlot数が増えていくので、初期に多くのSlotを確保しておく戦略の場合は、`RUBY_GC_HEAP_GROWTH_FACTOR`の値を下げておくのがよいかもしれない。
無論、プロセスのライフサイクルによる。

たとえば、Unicornで動くアプリケーションで、Out of Band GCと [unicorn-worker-killer.gem](https://github.com/kzk/unicorn-worker-killer)でOomKillerを有効にしてあって、RSSを一定量超えたworkerがkillされるような設定の運用だと、`RUBY_GC_HEAP_GROWTH_FACTOR`の倍率によってはすぐに規定量のRSSに達してすぐにworkerがkillされてしまうような事も起こりえる。
その場合にそなえて、`RUBY_GC_HEAP_GROWTH_FACTOR`の倍率は控えめに設定してworkerの寿命を延ばして、長いスパンで見たスループットを稼ぐ、という選択肢もあり得る。

この手のpreforkなworkerで動くRailsアプリケーションなどは、アプリケーションのコードがpreloadされた状態で`GC.stat`を確認し、使用されているslot数が、`RUBY_GC_HEAP_INIT_SLOTS`で確保しておく最低限のSlot数になるであろう。

バッチなど、ともかくメモリ使用量よりGCの停止時間を短くしたい場合は、`RUBY_GC_HEAP_INIT_SLOTS`や`RUBY_GC_HEAP_FREE_SLOTS`は多めにしておき、`RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR`も増やしておくことで、GC自体の実行回数を押さえることが出来ると予想される。

手元にある、ActiveRecordオブジェクトを大量にInstance化して利用するバッチでは、終了時に使用していたSlot数(`:heap_live_slot`)に`RUBY_GC_HEAP_INIT_SLOTS`を設定することで、約20%の速度改善を達成できた。

また、このバッチはrssを大量に(3, 4GB)ほど確保するので、`RUBY_GC_MALLOC_LIMIT`および`RUBY_GC_OLDMALLOC_LIMIT`を512MBに引き上げることで、かなりの効果を確認できた。

# 参考

+ [www.atdot.net/~ko1/activities/oedorubykaigi03_ko1_pub.pdf](http://www.atdot.net/~ko1/activities/oedorubykaigi03_ko1_pub.pdf)
+ [performance - Garbage collector tuning in Ruby 2.0 - Stack Overflow](http://stackoverflow.com/questions/16299419/garbage-collector-tuning-in-ruby-2-0)
+ [Watching and Understanding the Ruby 2.1 Garbage Collector at Work - Thorsten Ball](http://thorstenball.com/blog/2014/03/12/watching-understanding-ruby-2.1-garbage-collector/)
+ [[その1] Ruby 2.0のガベージコレクタを使いこなす - ワザノバ | wazanova](http://wazanova.jp/items/759)
+ [[その2] Ruby 2.0のガベージコレクタを使いこなす - ワザノバ | wazanova](http://wazanova.jp/items/761)
+ [RGenGCに対応する方法 - mirichiの日記](http://d.hatena.ne.jp/mirichi/20131010/p1)
+ [第5章 ガ－ベージコレクション](http://i.loveruby.net/ja/rhg/book/gc.html)
+ [www.is.titech.ac.jp/~sassa/lab/papers-written/08M37090-oota.pdf](http://www.is.titech.ac.jp/~sassa/lab/papers-written/08M37090-oota.pdf)
+ [RubyとGCについて - mirichiの日記](http://d.hatena.ne.jp/mirichi/20130611/p1)

