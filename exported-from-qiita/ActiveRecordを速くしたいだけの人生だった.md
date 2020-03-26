---
title: ActiveRecordを速くしたいだけの人生だった
tags: Ruby Rails4 rails3.2
author: yuroyoro
slide: false
---
Rails3.2からRails4.2に上げたらActiveRecordが遅くなったので、どうやって調査して、どのように対処したかを語ってみたい。

とても長いので、ダルい人は最初と最後だけ読めばよいです。

# TL;DR

環境:

+ Ruby 2.1.5
+ ARオブジェクトを大量に(ざっくり750kくらい)loadするバッチ処理
+ 3.2系での実行時間は約480sec、 4.2系では約2900sec
+ 約6倍の性能劣化

原因:

+ `preload`で性能劣化してた
+ `CollectionProxy`の生成周りで遅くなってた
+ Rails4からARオブジェクトの1attribute毎にObject生成するので遅い
+ GCの時間も増えた

調査方法:

+ Githubのcommit、Issueを漁った
+ `GC Profiler`でGCの状況を確認しましょう
+ `benchmark-ips`でベンチとりましょう
+ `ruby-prof` と `kcachegrind`、 `stackprof`と`stackprof-webnav` などでプロファイルしましょう

対処:

+ `preload`の劣化はPRだした
+ `CollectionProxy`はmokey patchで対処
+ attribute毎にObject生成しないように3.2系の処理をbackport
+ jemalloc入れた
+ Ruby2.2.1まであげた
+ 上記の対処で約580secまで回復した

# 経緯

手元にあるRailsアプリケーションを3.2系から4.2までupgradeしたところ、ActiveRecordオブジェクトを大量にメモリに読み込むバッチ処理の性能が著しく劣化した。
3.2系で530secだった実行時間が2900secまで伸び、このままではproductionに投入できない状況だった。

この性能劣化自体は約1年ほど前から把握しており、システムにとって非常に重要なバッチ処理なので4系への移行を見合わせていた。
このあたりのツイートに苦悩がにじみ出ている。

<blockquote class="twitter-tweet" lang="en"><p>【緩募】Rails4に移行して急速にARのパフォーマンスが急速に劣化した経験をお持ちの人。およびその対策 F.Y.I -&gt; Rails 4 preload performance - Google グループ <a href="https://t.co/wjkov0VIXz">https://t.co/wjkov0VIXz</a></p>&mdash; ⁰⁰⁰⁰null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/448026482946736128">March 24, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
<blockquote class="twitter-tweet" lang="en"><p>Rails4、人類にはまだ早すぎる</p>&mdash; ⁰⁰⁰⁰null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/448040819467907072">March 24, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
<blockquote class="twitter-tweet" lang="en"><p>Rails4でのActiveRecordのearger loadingで20%ほどパフォーマンス劣化してる。ほかにもARがかなり遅くなっており辛み。4.1rcでも改善せず</p>&mdash; ⁰⁰⁰⁰null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/448379276048363521">March 25, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
<blockquote class="twitter-tweet" lang="en"><p>Rails4被害者の会</p>&mdash; ⁰⁰⁰⁰null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/453747739751243778">April 9, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


ところが、Rails3.2系ではRuby2.2系はサポートされない、という状況になり、3.2系にこだわるとRuby自体のバージョンも上げることができなくなる。これが、Rails4系への移行に本気で取り組むことにした最大の理由だ。

<blockquote class="twitter-tweet" lang="en"><p>Rails4系へ移行したい理由:&#10;&#10;- Rails3.2系はSecurity patch以外入らない&#10;- Ruby2.2以降のサポートがなさそう&#10;- 依存Gemのメンテがされなくなりつつある&#10;- ゆるやかに腐っていく</p>&mdash; ⁰⁰⁰⁰null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/557370438477565952">January 20, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

# 調査

性能劣化の原因はどこにあるのか、調査はまずアプリケーションのログを解析するところから始めた。ここで判明した事実は以下の通り。

+ DBからSELECTしてActiveRecordオブジェクトとしてloadする箇所が大きく性能劣化している
+ そのほかの処理でも全体的に劣化している

この段階では、正直どこから手をつけたものか途方にくれていた。

## Profileを取る

[「推測するな、計測せよ」](http://ja.wikipedia.org/wiki/UNIX%E5%93%B2%E5%AD%A6)の言葉どおり、まずはProfileを取ってみる。


Rubyでのprofilingについては、`ruby-prof`による測定と、`stackprof`を用いた方法がある。
まずは`ruby-prof`を使ってみた。
stackprof`を用いた方法については後述する。

### ruby-profでprofile

`ruby-prof`は結構昔からあるツールで、結果をKCacheGrindというツールでビジュアルに確認できる。が、Profileの取得はそれなりにオーバーヘッドがある。
`ruby-prof`については、まずはこのエントリを読むべし。

[ruby-profとKCacheGrindでプロファイル野郎になる - 昼メシ物語](http://blog.mirakui.com/entry/20100919/rubyprof)

さて、実際にRails3.2とRails4.2でそれぞれProfileを取得してみて、傾向を比較してみることにした。

Rails3
![ruby_prof_rails3.png](https://yuroyoro.github.io/exported-from-qiita/images/c6451391-5b6a-dc19-3d53-e6a3fb57b929.png)

Rails4
![ruby_prof_rails4.png](https://yuroyoro.github.io/exported-from-qiita/images/b105775f-fe3a-88d0-4ae9-1c8312e4e2be.png)


上の画像は、取得したProfileをqcahegrindに喰わせて、self(そのメソッドのみの処理時間の合計)でソートした結果である。
Rails4.2では、3.2には登場しなかった`ActiveRecord::Attributes`のメソッドが大量に呼び出されていることが判明した。


## GithubでIssueを漁る

とりあえずGithubのIssueにそれっぽいのが上がっていないか漁ってみると、以下のIssueが見つかった。

[Introduce an Attribute object to handle the type casting dance by sgrif · Pull Request #15593 · rails/rails](https://github.com/rails/rails/pull/15593#issuecomment-45601323)

このPRの内容は、Rails3.2におけるデータベースとActiveRecordオブジェクトの間の型変換は、ではメタプログラミングを駆使した見通しの悪い実装になっていたので修正した、というもの。`ActiveRecord::Attributes`はこのPRで導入されている。
PRの議論の中で、Objectのアロケーションについて議論があるが、各ARオブジェクトひとつのカラムに対して、ActiveRecord::Attributesのインスタンスがひとつ生成される、とある。


もうひとつ、関係のありそうなIssueがある。

[Major performance regression when preloading has_many_through association · Issue #12537 · rails/rails](https://github.com/rails/rails/issues/12537)

`:has_many_through`な関連を`preload`で指定してloadすると、レコード数に比例して性能劣化するという内容だ。これも原因のひとつに見える。

## Issue #12537について調べる

`ActiveRecord::Attributes`の件はなんか大変そうなんで後回しにして、Issue #12537について調べてみることにした。もとのIssueに再現するためのコードが添付されているので、これを用いて`stackprof`でProfileしてみることとする。

### stackprofでProfile

[tmm1/stackprof](https://github.com/tmm1/stackprof) は、Ruby2.1から利用できる`rb_profile_frames()`というC-APIを用いた軽量なプロファイラーで、作者はGithubの中の人の[tmm1 (Aman Gupta)](https://github.com/tmm1)。
特徴は、プロファイリングのオーバーヘッドがほとんどないことで、実際にproduction環境でも取得できるらしい。
以下のの記事は非常にわかりやすいので必読。

[Ruby 2.1: Profiling Ruby · computer talk by @tmm1](http://tmm1.net/ruby21-profiling/)


Profilingの準備として、Githubから[rails/rails](https://github.com/rails/rails) をcloneして、手元の環境で弄れるようにしておく。
cloneしたあとに、`bundle install`して依存をいれる。
そして、`bundle exec rails new ../ar-prof-app --edge --dev`で、 cloneしたRailsを利用するテスト用アプリケーションを生成しておく。
このテスト用アプリケーション内で、サンプルのベンチマークを実行してプロファイルを取得する。
実際にプロファイルを取るのに利用したスクリプトがこれ。

[profile_preloading.rb](https://gist.github.com/yuroyoro/94e55bd05e748b052689)

さて、`stackprof`で取得した結果を解析に入るのだが、`stackprof`の出力を元にコード上のどの行がどの程度の実行時間を占めているのか可視化してくれる [stackprof-webnav](https://github.com/alisnic/stackprof-webnav)という非常に便利なツールがあって、これに大変お世話になった。

使い方はGithubのREADMEを見てもらえればすぐにわかると思う。
Issue #12537でtenderloveが、[ActiveRecord::Associations::Preloader::ThroughAssociation#associated_records_by_owner
](https://github.com/rails/rails/blob/19b871a0902c4ec3e460a38f41583a7855edd81a/activerecord/lib/active_record/associations/preloader/through_association.rb#L13) に手を入れているので、ここを中心に見ていくこととする。

![stackprof_1.png](https://yuroyoro.github.io/exported-from-qiita/images/02d5e40f-4beb-122c-52f9-618503d9f7b4.png)


`ActiveRecord::Associations::Preloader::ThroughAssociation#associated_records_by_owner`をクリックすると、calleeにそこから呼出しているメソッドが表示される。この中で実行時間の多くを`ActiveRecord::Associations::collectionassociation#reader`メソッドが占めていることがわかるので、さらに辿っていく。

![stackprof_2.png](https://yuroyoro.github.io/exported-from-qiita/images/820b520f-bfbd-f558-4ff1-090c2d39842c.png)


`ActiveRecord::Delegation::ClassMethods#create` -> `ActiveRecord::Associations::CollectionProxy#initialize` まで到達した。どうやら`CollectionProxy`のインスタンス化はコストの高い処理のようだ。

`:has_many_through`をloadする際にレコード数分この`CollectionProxy`を生成していることが原因に見える。

実際、`associated_records_by_owner`メソッドでは`CollectionAssociation#reader`を2回呼び出しており、この2回の呼びだしがメソッド実行時間中の7割を占めていることがプロファイル結果から読み取れる。

![stackprof_3.png](https://yuroyoro.github.io/exported-from-qiita/images/49feb9e3-a79a-2448-74cf-f53aba1a9685.png)



`CollectionProxy`の生成は、`preload`以外にassociationをたぐると行われるので、その他の場所でも性能に影響を与えている可能性が高い。


### 再びGithubを漁る

プロファイル結果から、`CollectionProxy`の生成が遅いことが判明した。しかし、Rails3で試してみると、`CollectionProxy`の生成にはほとんど時間はかからない。
となると、Rails4でこのクラスに何らかの変更が入ったと見るべき。
commit logを見ると、興味深いコミットを見つけた。

[CollectionProxy < Relation · rails/rails@c86a32d](https://github.com/rails/rails/commit/c86a32d7451c5d901620ac58630460915292f88b#commitcomment-5384529)

どうも、Rails4では`CollectionProxy`は`Relation`のサブクラスになったようだ。Rails3では、単なるdelegateするだけのクラスだったが振る舞いが変わったようだ。コミットのコメントをみると、tenderloveが「なんか遅くなってる」みたいなコメントと[関連するIssue](https://github.com/rails/rails/issues/14033) を上げていて、「解決したよ」みたいな雰囲気だしてるけど、すいません直ってないです……。

## GCを調べる

`ActiveRecord::Attributes`の導入によってARの1レコードの各カラム毎にオブジェクトが生成されているのなら、GCの実行時間が増加しているはず。そのように思って、じっさいのバッチアプリケーションでのGC実行時間をRails3.2系と4.2で比較してみることにした。

`GC.stat`を用いると、実行回数などの統計情報が取得出来る。これについては以前に[Ruby 2.1.1 GC Tuning - Qiita](http://qiita.com/yuroyoro/items/14ec7079f6574ad74409)で書いた。

さらに、標準添付されている[GC::Profiler](http://docs.ruby-lang.org/ja/2.0.0/class/GC=3a=3aProfiler.html)を用いることで、実行回数や時間などの情報を得ることができる。以下は出力の例。

```
[GC::Profiler.result]
GC 22 invokes.
Index    Invoke Time(sec)       Use Size(byte)     Total Size(byte)         Total Object                    GC Time(ms)
    1              15.218             34180560            959991360             23999784       858.67890400000135286973
    2              21.655             41330880            959991360             23999784       509.01075000000162162905
    3              26.315             40650880            959991360             23999784       612.90844099999833360926
    4              30.784             42514320            959991360             23999784       453.21988099999896348891
    ...
```

結果は、Rails3ではGC実行回数 22、実行時間 37400msに対して、Rail4ではGC実行回数 34、実行時間105700ms となった。GCだけで約70秒増えていることになる……。


## ここまでの調査結果

ここまでの調査で判明した事実。

+ `ActiveRecord::Attributes` の導入で性能劣化(っぽい?)
+ 関連してGCの実行回数、時間が増加
+ `:has_many_through` を大量レコードで`preload`すると遅い
+ `CollectionProxy`の生成が遅い

こうして書いてみると、さくっとプロファイルしてIssue調査して速やかに原因特定したように読めるが、実際にはプロファイルの仕方を変えたりログをあちこちに仕込んだりqcahegrindを眺め廻して推測したりと、1週間以上の間、試行錯誤を繰り返してかなり苦労してる。プロファイル結果を正確に読み取ったり当たりをつけたりするスキルが必要ですな。


# 対処

原因が判明したところで、どうにかこれらに対処して性能を元に戻せないか格闘してみた。

## Issue #12537 を直す

まずは、`preload`の性能劣化について修正を試みる。プロファイルにより、`CollectionAssociation#reader`の呼び出しが重いことが判明しているので、これをどうにかできればいい

### ベンチマークとる

まずは、修正に着する前にベンチマークスクリプトを書いてみた。

[benchmark_preload_has_many_through.rb](https://gist.github.com/yuroyoro/89fae1150e285658db45)


ベンチマークには、[benchmark-ips](https://github.com/evanphx/benchmark-ips)を用いるべし、と[Contributing to Ruby on Rails — Ruby on Rails Guides](http://guides.rubyonrails.org/contributing_to_ruby_on_rails.html)にある。

`benchmark-ips`は、単位時間当たりの実行回数(と標準偏差)で比較できるので、この数値が大きいほど速い、ということになる。


3.2系でのベンチマーク結果は以下の通り。一番下の`limit 1000 double has_many_through`で3.52回/秒。
最新のmasterで、0.516回/秒。

```
================================================================================
3.2.21 - postgresql
================================================================================

Calculating -------------------------------------
  limit 100 has_many    12.000  i/100ms
limit 100 has_many_through
                         5.000  i/100ms
limit 100 double has_many_through
                         3.000  i/100ms
 limit 1000 has_many     1.000  i/100ms
limit 1000 has_many_through
                         1.000  i/100ms
limit 1000 double has_many_through
                         1.000  i/100ms
-------------------------------------------------
  limit 100 has_many    129.401  (± 3.9%) i/s -    648.000
limit 100 has_many_through
                         53.440  (± 7.5%) i/s -    270.000
limit 100 double has_many_through
                         31.399  (± 6.4%) i/s -    156.000
 limit 1000 has_many     15.410  (± 6.5%) i/s -     77.000
limit 1000 has_many_through
                          6.097  (± 0.0%) i/s -     31.000
limit 1000 double has_many_through
                          3.520  (± 0.0%) i/s -     18.000

================================================================================
5.0.0.alpha - postgresql
================================================================================

Calculating -------------------------------------
  limit 100 has_many     8.000  i/100ms
limit 100 has_many_through
                         1.000  i/100ms
limit 100 double has_many_through
                         1.000  i/100ms
 limit 1000 has_many     1.000  i/100ms
limit 1000 has_many_through
                         1.000  i/100ms
limit 1000 double has_many_through
                         1.000  i/100ms
-------------------------------------------------
  limit 100 has_many     92.615  (± 4.3%) i/s -    464.000
limit 100 has_many_through
                          8.574  (± 0.0%) i/s -     43.000
limit 100 double has_many_through
                          5.300  (± 0.0%) i/s -     27.000
 limit 1000 has_many     12.712  (± 0.0%) i/s -     64.000
limit 1000 has_many_through
                          0.892  (± 0.0%) i/s -      5.000  in   5.610751s
limit 1000 double has_many_through
                          0.516  (± 0.0%) i/s -      3.000  in   5.814872s

```


### 修正する

```ruby
        def associated_records_by_owner(preloader)
          preloader.preload(owners,
                            through_reflection.name,
                            through_scope)

          through_records = owners.map do |owner|
            association = owner.association through_reflection.name

            center = target_records_from_association(association)
            [owner, Array(center)]
          end
```

つまり、速くするには`CollectionAssociation#reader`を呼び出さなければよい。なぜこのメソッドを呼び出す必要があるかというと、`:has_many_through`の中間レコードを取得する必要があるからなのだが、実は`reader`を呼び出さずとも、すでに中間レコードはloadされているので、`CollectionAssociation#target`メソッドを呼び出すことで、重い`CollectionProxy`のインスタンス化を避けて中間レコードを取得できる。

`targe`メソッドを利用したバージョンで、2.144回/秒。3.2系の速度には及ばないが、かなり改善した。

```
================================================================================
5.0.0.alpha (after) - postgresql
================================================================================

Calculating -------------------------------------
  limit 100 has_many     8.000  i/100ms
limit 100 has_many_through
                         3.000  i/100ms
limit 100 double has_many_through
                         1.000  i/100ms
 limit 1000 has_many     1.000  i/100ms
limit 1000 has_many_through
                         1.000  i/100ms
limit 1000 double has_many_through
                         1.000  i/100ms
-------------------------------------------------
  limit 100 has_many     91.012  (± 4.4%) i/s -    456.000
limit 100 has_many_through
                         31.822  (± 3.1%) i/s -    159.000
limit 100 double has_many_through
                         19.298  (± 5.2%) i/s -     97.000
 limit 1000 has_many     10.621  (± 0.0%) i/s -     54.000
limit 1000 has_many_through
                          3.767  (± 0.0%) i/s -     19.000
limit 1000 double has_many_through
                          2.144  (± 0.0%) i/s -     11.000
```

で、修正してPRをだした。

[Fix #12537 performance regression when preloading has_many_through association by yuroyoro · Pull Request #19423 · rails/rails](https://github.com/rails/rails/pull/19423/files)

けど、絶賛放置中(´・ω・`)。

## CollectionProxyのインスタンス化を速くする

Issue #12537の修正を実際のアプリケーションに適用して測定してみたが、思ったより速度が改善しなかった。
次に、CollectionProxyのインスタンス化をどうにか速くできないか試してみた。


### ベンチマークとる

例によって、まずはベンチマーク。

[benchmark_instantiate_collection_proxy.rb](https://gist.github.com/yuroyoro/16df3fa6458c1c768c1e)

```
================================================================================
Benchmark : Instantiate ActiveRecord::Associations::CollectionProxy
3.2.21 - postgresql
================================================================================

Calculating -------------------------------------
            has_many    78.000  i/100ms
    has_many through    77.000  i/100ms
has_many through through
                        77.000  i/100ms
-------------------------------------------------
            has_many    806.848  (± 1.1%) i/s -      4.056k
    has_many through    790.381  (± 0.9%) i/s -      4.004k
has_many through through
                        794.065  (± 1.3%) i/s -      4.004k

================================================================================
Benchmark : Instantiate ActiveRecord::Associations::CollectionProxy
4.2.0 - sqlite3
================================================================================

Calculating -------------------------------------
            has_many     1.000  i/100ms
    has_many through     1.000  i/100ms
has_many through through
                         1.000  i/100ms
-------------------------------------------------
            has_many     14.336  (± 0.0%) i/s -     72.000
    has_many through      8.100  (± 0.0%) i/s -     41.000
has_many through through
                          5.990  (± 0.0%) i/s -     30.000


```

Rails3.2系で794.065i/sのところが4.2系では5.990i/s……。遅いってレベルじゃねーぞ

### 修正してみる

Profileの結果、`CollectionProxy`の生成時には`Association#scope`を経由してProxy先のAssociationから`Relation`オブジェクトを生成し、それを自身に`merge`することが劣化の原因と判明した。

なぜこの処理が必要かというと、`CollectionProxy`が`Relation`クラスのサブクラスとして振る舞うために、Proxy先の`Relation`オブジェクトの条件(whereとかorderとか諸々)を自身に`merge`メソッドでマージする必要があるから。

しかし、実際に自身のインスタンス変数としてProxy先の条件を記憶せずとも、全てのメソッド呼び出しを`Association#scope`にdelegateしてしまう形でも、`Relation`クラスのサブクラスとしての振る舞いになる。実際に、Rails3.2時代の`CollectionProxy`は単に他のオブジェクトへ(条件次第で)delgateしているだけだった。
そのようにして修正したコミットがこれ。

[Delegates all Relation's method to scope · yuroyoro/rails@92980bf](https://github.com/yuroyoro/rails/commit/92980bf16cb75cae7d52695ff5aef0b2921cdf76)

ベンチマークの結果、629.406i/sまで回復した。

```
================================================================================
Benchmark : Instantiate ActiveRecord::Associations::CollectionProxy
4.2.1.rc3 - postgresql
================================================================================

Calculating -------------------------------------
            has_many    61.000  i/100ms
    has_many through    60.000  i/100ms
has_many through through
                        61.000  i/100ms
-------------------------------------------------
            has_many    629.340  (± 1.3%) i/s -      3.172k
    has_many through    612.921  (± 0.8%) i/s -      3.120k
has_many through through
                        629.406  (± 0.8%) i/s -      3.172k
```

いちおうテストは通っているが、実装としてトリッキーなうえに、設計としても妥当か判断がつかないし、英語でこのあたりの議論ができる自信がないのでPRは出していない……。

この修正により、対象のバッチの実行時間が約2900sec から1900secまで改善した。が、まだ遅い……。

## ActiveRecord::Attributes問題

プロファイルから、`ActiveRecord::Attributes`の導入によって、1レコードの1カラム毎にオブジェクトが生成されてオーバーヘッドが生まれている事が判明した。
さらに調査すると、string型のカラムをreadする際に性能劣化していることがわかった。

### ベンチマーク


念のため、単にARオブジェクトをloadしたのち、string型のカラムをreadしてみるだけのベンチマークを書いて確認してみる


[benchmark_compiled_method.rb](https://gist.github.com/yuroyoro/4953e7b7e57479b844d6)


```
================================================================================
Benchmark : ActiveRecord read by compiled attribute method (string only)
3.2.21 - postgresql
================================================================================

Calculating -------------------------------------
     compiled method    32.000  i/100ms
-------------------------------------------------
     compiled method    332.114  (± 1.8%) i/s -      1.664k

================================================================================
Benchmark : ActiveRecord read by compiled attribute method (string only)
4.2.0 - postgresql
================================================================================

Calculating -------------------------------------
     compiled method    21.000  i/100ms
-------------------------------------------------
     compiled method    219.852  (± 0.9%) i/s -      1.113k
```

Rails4.2系では、string型のカラムの読み出しが2/3の速度になっている。今回のバッチアプリケーションでは大量のstring型カラムを読み出すので､無視できないオーバーヘッドである。

### 修正してみたい


色々と試行錯誤してみたが、read時には`ActiveRecord::Attributes`を生成せずに、Rails3.2系の処理をbackportするしか方法が思いつかなかった……。

[Avoid instantiate ActiveRecord::Attribute · yuroyoro/rails@cb42fbb](https://github.com/yuroyoro/rails/commit/cb42fbb9488cc7ec2412541be62f7a4dfa2d4922)


```
================================================================================
Benchmark : ActiveRecord read by compiled attribute method (string only)
4.2.1.rc3 - postgresql
================================================================================

Calculating -------------------------------------
     compiled method    43.000  i/100ms
-------------------------------------------------
     compiled method    435.002  (± 0.7%) i/s -      2.193k

```

修正の結果、バッチの実行時間が約630secまで改善した。やったー！！
でもこれでいいのか……?

とりあえず、これらの修正をシステム全体に適用するのはあまりに危険なので、monkey patch化して必要なバッチ処理だけloadするようにした。

## Ruby2.2.1まであげてjemalloc入れた

Ruby2.2ではSymbolGCやインクリメンタルGCが実装されており、バージョンをあげることで高速化が期待される。
さらに、2.2からは`configure --with-jemalloc`で`jemalloc`をビルド時にlinkできるようになったので、Ruby 2.2.1 with jemalloc で測定してみた。

結果、バッチの処理時間が約580secまで改善。なかなかの効果だった。

# 結論

+ いろいろと危ないmonkey patchを書くことでRails4でもどうにか性能がでるようになった
+ Railsで速度が重要なバッチ処理を実装するのはあまりおすすめできない
+ [「ScaleOutではedge railsと闘いたいエンジニアを募集しています」](http://www.scaleout.jp/about/job-opportunities/)

