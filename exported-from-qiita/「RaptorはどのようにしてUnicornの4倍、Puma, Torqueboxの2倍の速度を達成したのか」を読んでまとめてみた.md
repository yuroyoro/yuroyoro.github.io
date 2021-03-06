---
title: 「RaptorはどのようにしてUnicornの4倍、Puma, Torqueboxの2倍の速度を達成したのか」を読んでまとめてみた
tags: Ruby rack httpd
author: yuroyoro
slide: false
---
「RaptorはどのようにしてUnicornの4倍、Puma, Torqueboxの2倍の速度を達成したのか」を読んでまとめてみました。

原文はこちらです。紹介については許可を貰っています。

[How we've made Raptor up to 4x faster than Unicorn, up to 2x faster than Puma, Torquebox](http://www.rubyraptor.org/how-we-made-raptor-up-to-4x-faster-than-unicorn-and-up-to-2x-faster-than-puma-torquebox/)

とても読みやすい英語ですので是非原文も読んでみてください。

## How Ruby app servers work

Rackアプリケーションの構成についての紹介と、コネクションをどのように扱うのかについて。
prefork/threadingやBlocking I/OおよびEvent I/Oの組み合わせとSlow Client問題について。

* RaptorのI/Oモデルは後述のbuffering reverse proxyとunicornのようなpreforkの組み合わせ。
* 有料でmulti threadサポート

## Writing a fast HTTP server

RaptorではRackアプリケーションサーバーの前段に、C++で書かれたreverse proxy用HTTP Serverを配置するらしい。
Rackアプリケーションに特化しており、静的ファイルの配信やgzip圧縮は無いが、その分Nginxの2倍のスピードとのこと。


### The Node.js HTTP parser

HTTP parserはnode.jsのを使用。keep aliveやupgradeやchunkに対応し、高速でzero copyなので。セキュリティ面も考慮

* MongrelのparserはRagelで生成されていて、ThinやUnicornでも使用されていて信頼性もあるが、Ruby以外で使うのが辛み
* H2OのPicoHTTPParserは実績不足

### The libev event library

Event I/Oのためにlibevを使用している。libeventより機能はすくないがその分高速なため

### Hybrid evented/multithreaded: one event loop per thread

Raptorでは、EventベースのI/OとMulti Threadingを組み合わせている。
CPUコアを使い切るためHTTPServerはcore数だけthreadを起こしてそれぞれでevent loopを持つ。これにより、
接続の遅いクライアントにも対処できる。

この形式だと、kernelのschedulingによって、最初のThreadに接続が集中する問題が発生した。
(筆者:irq balancingのことだろうか?)

この問題の対処のために、内部的に利用するround robinなload balancerを書いて特定のthreadに処理が偏らないようにした。

### Zero-copy: reducing CPU working set,  coping with memory latencies

HTTPServerは、I/O操作のためのbufferを管理する必要がある。このbufferはCPUのL1 Cacheに比べて大きいので、都度copyが行われるとCPU Cacheを汚染してしまう。

RaptorのHTTPServerは、CPUのcacheを有効に使い切るために、必要になるまでメモリのコピーを行わないようにする"zero-copy architecture"で構成されている。

このアーキテクチャは、mbufsとscatter-gather I/Oの2つのサブシステムで構成される。


### Mbufs: reference-counted,  heap-allocated,  reusable memory buffers

mbufsという参照カウント式のアロケーター。twemproxyを元にしており、 I/O操作用のbuffer専用に使用される。

read(2)で読んだdataは呼び出した関数のstack上のbufferに詰まれている。Blockin I/OではI/O操作が完了するまでblockするので、stack上に配置されていても問題無いが、Event I/Oでは非同期のため、呼び出した関数のstackのscopeを超えてbufferを保持しておく必要がある。そのため、通常はstackからcopyして待避しておく。

mbufsでは、clientからの接続毎にHeapからallocationして、requestが終わったら解放せずにfreelistに戻す。次回は、freelistにあればそこから再利用する。

* 参照カウントの管理はRAII
* Node.jsライクなbuffer sliceもサポート。Bufferの一部を参照可能
* Twemproxyのやつはグローバル変数使ってたlockを獲ってたので性能劣化した。thread毎にcontextを与えてthread-safeにした。
* bufferの使用状況を参照する管理機能

デメリットとして、確保したmemoryをosに返さない。peak時にmemoryが枯渇する可能性があるが、管理ツールでosに返す事が可能。
また、Thread間をまたいだmemoryの共用はできない

### Scatter-gather I/O: avoiding concatenation of buffers

メモリ上で分散されている巨大な文字列をsocketに書き込む場合、2通りの方法が考えられる。

* 結合してからkernelに渡す -> メモリのcopyが発生する
* それぞれでシステムコール -> システムコール多発

Raptorではwritev(2)を使って、文字列の配列をkernelに渡すことで、一度のシステムコールでzero copyで実現している。

### Avoiding dynamic memory allocations


C++でのnewによる動的allocationは高コスト。 allocateとfreeは実は複雑なオペレーションであり、
allocatorにもよるが、thread間での共有のためにlockを獲ることがある。
tcmallocではthread毎にcacheを持つが、完全にこれを回避できるわけでは無い。

動的なallocationは避けるために、 object poolingとregion based memory managementの二つのテクニックを用いる


### object pooling

Object poolingはmbufsに似た方式で、Objectが不要になれば単にfreelistに戻し、必要に応じて再利用する。

boost::poolを利用して、事前に用意されている隔離された領域を小分けにして使う。 
objectのsizeに応じて分割されるので、CPU cacheのlocalityが向上する、 mallocのようにblock sizeでallocateすることによる無駄な領域のオーバーヘッドがない、などの利点がある。

Raptorでは、RequestやHttpParserやSessionなど、頻繁に作成されるobjectをpoolすることで多大な効果を得ている。
poolingすることにより、省メモリで必要に応じたObjectを動的に確保することができるようになった

### Palloc: stack-like, region-based memory management

Object Poolingは固定長だが、場合によっては様々な可変長のメモリ領域がほしいこともある。Stackへのallocationは、Event I/Oのライフサイクルでは問題になることもある。

そのため、 小規模な可変長データ構造向けのアロケータを用意した

* Requestに関連した16 KBのchunkを用意しておき、小規模なmemoryのためにStackのように使う(例えば日付のforamat用とか)
* allocationはpointerを操作するだけなので高速
* pallocのstack pointerはdecrementしない。allocateはできるが解放はできない。requestの処理が終わったらpointerは0にresetされ、再利用可能となる
* 16KB以上必要になったら別のchunkがallocateされる
* Nginxのものをベースにしているが、Nginxと異なりdeallocateせずに再利用する
* pallocで確保した構造は、stackのようにCPU Cacheへのローカリティが非常に高い。
* デメリットとしては、一度確保したらrequestの処理が終了するまで解放されないこと


### 筆者まとめ

可能な限りメモリのcopyやallocation回数を減らすため、用途に応じたアロケーターを用意して使い分けている。
メモリのライフサイクルも、リクエスト処理に紐付けることで、Thread内でContextが限定され、解放するタイミングも制御しやすい。

ソースコードはまだ公開されていないが、公開されたら読んでみようと思う。

