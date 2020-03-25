---
title: Rails5.1から導入されるwebpacker.gemは本当にRailsのフロントエンド開発に福音をもたらすのか?
tags: Rails Rails5 webpack
author: yuroyoro
slide: false
---
Rails5.1が今betaで出ていますね。中でも目玉はwebpacker.gemによるモダンなフロントエンド開発がRailsに導入されることでしょう。
今までのRailsのasset pipelineとは別に、yarnによって依存性を管理しwebpackで結合する独立したjsのビルドシステムがサポートされます。
これによって、以下のような従来のasset pipelineでは解決がむずかしかった問題への解が示されました。

- coffee scriptへの依存
- npmによる依存性、バージョン管理が難しい
- javascriptのライブラリが野良gem化されてupdateされない問題

webpacker.gemはyarn/webpackの薄いwrapperとなっていて、加えて幾つかのrakeタスクを追加することでフロントエンド開発をサポートします。
具体的には以下のような機能が提供されます。

- yarn/webpackのbinstub
- babelによるtranspile
- webpackによる生成物はpublic/packs以下に配置
- webpacker:installによる環境立ち上げ
- React, vue.js, angular2のサポート
- webpack-dev-serverによるhot reloading
- ビルド生成物へのdigestの付与とヘルパータグ
- erb.jsをwebpackで結合する rails-erb-loader

チュートリアル動画や紹介記事をみてもらえばわかりますが、コマンド叩けばさくっとフロントエンド開発環境が立ち上がってすぐにes6でjavascriptも書けて、dev-server起動すればhot reloadingもできて、この立ち上がりの速さはいかにもRailsといった感じです。
でも、これで本当にRailsとフロントエンド開発に関わる問題が解決されて、幸福の時代が訪れるのでしょうか?
確かに、冒頭にあげた3つの問題は解決されますが、他に問題はおきないのでしょうか?

筆者が現在のプロジェクトでRails5.0 + webpacker.gemを導入して発生した問題点とどのように対処したかを紹介し、本当にこれでよいのかを考えてみたいと思います。(使用したwebpacker.gemは2017/02/04 4405364b8c951cfd84fe9a12f35b2e45bd3d9e9d です)


## cssのビルドについて

まずwebpacker:install で生成されるwebpack.configでは、基本的にjsのビルドしかできないようになっています。yarnでinstallされたライブラリに含まれるcssやimage, fontなどは、css-loaderを利用してjsにpackするしかありません(あとは独自管理か…)。
scssを使って、jsとは別にcssもwebpackでビルドしてpackしたい、という場合は一手間かかることになります。

筆者はwebpacker.gemが生成するwebpack.configを書き換えて、jsとは別にscssをビルドできるように修正しました。

## entrypointの肥大について

webpacker.gemでは、 app/javascript/packs/ 以下のjsファイルがすべてentrypointとして扱われます。よって、いままでのようにコントローラーごとにpacks以下にjsをおいておくと、多数のjsファイルが生成されます。
ここで、各jsファイルがそれぞれ大きめのライブラリをrequireしてしまうと、それぞれのビルド成果物のファイルサイズが肥大化してしまいます。

これに対処する方法はいくつかあります。ひとつは、コントローラーごとにjsを書かずにひとつのjsファイルに結合して、client側でなんらかのルーターを導入する方法。もう一つは、`webpack.optimize.CommonsChunkPlugin` でライブラリだけ別なjsファイルに纏めてしまう方法です。今回は後者でやりました。

あと、app/javascript/packs以下のサブディレクトリはentrypointにしてくれないので、それもwebpack.configを書き換えて対処しました。

## ディレクトリ構成について

2017/03/01時点の最新バージョンでは、 Rails.root以下にnode_modulesとpackage.jsonが配置されるようになっています。

~~webpacker.gemが期待するディレクトリ構成がなかなか独特で、妙な位置( `vendor` 以下) にnode_modulesとpackage.jsonが配置されています。Rails.rootで作業していて、yarnでインストールしたeslintコマンドなどを叩こうとすると、毎回package.jsonやnode_modulesの位置を引数につける必要があったりして面倒です。個人的には、Rails．root以下にfrontendなどというディレクトリを用意して、jsもconfigもnode_modulesもすべてそのディレクトリに隔離してしまえばよかったように思えます。~~

```
.
├── app
│   └─── javascript
│             └── application.js
├── config
│   └── webpack
│             ├── development.js
│             ├── production.js
│             └── shared.js
├── public
│   └── packs
├── node_modules
├── package.json
└── yarn.lock
```

## digest付与について

そもそもdigest付与って必要性が減ってきている気がします。CDNを使って配布するならそもそも不要だし。webpacker.gemでのdigest付与は、webpack側で生成されたhashを public/packs/digest.json に書き込んで、Rails側の javascript_pack_tag はこのjsonを手がかりにファイル名を解決しています。
この仕組自体はそんなに難しいものじゃないので、独自に実装してもよいでしょう。実際に、cssもビルドできるようにwebpack.configを書き換えたので、そのに対応したdigestを生成するようにrakeタスクを書き直しました。

## で、結局どうだったの?

webpacker.gemが提供するフロントエンド開発環境は、必要なものがひととおりそろっていてすぐに開発を始めることができる、という点ですばらしいと思います。
いっぽう、提供されるwebpack.configやディレクトリ構成は制約が多く、フロントエンドエンジニアが自由にできる余地は少ないです。
あくまで、「Railsエンジニアがフロントエンド開発をすぐにできる」ことにフォーカスしていて、完全なフロントエンドとの分離ではないです。

「webpacker.gemが提供するレール」に乗って開発する分には十分快適ですが、レールから外れた場合は足かせが多いです。
よって、フロントエンド側を独立させて自由にやりたい場合は、webpacker.gemを使わないでビルドし、生成物をpublic以下やCDNで配布するほうがよいでしょう。
(たとえば、 こういうやりかたでも十分だと思います -> [webpackで作るSprockets無しのフロントエンド開発 \- クラウドワークス エンジニアブログ](http://engineer.crowdworks.jp/entry/2016/05/24/174511) )

レールから外れない限りは、オフィシャルのサポートがあってyarn管理できるという点で、楽ができそうです（あくまでレールに乗り続ける限りは……)。
sprocketsとasset pipelineは悪い文明なのでさっさと捨てましょう。
とはいえ、妙なディレクトリ構成とか、中途半端にerbをサポートしたりとか、歪な点も多いです。

<blockquote class="twitter-tweet" data-lang="en"><p lang="ja" dir="ltr">つまりレールズはなんでもレールズの一部として歪な形で取り込むのをやめろ <a href="https://t.co/oRxC56jEWu">pic.twitter.com/oRxC56jEWu</a></p>&mdash; null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/831685757655883776">February 15,  2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

最後に、今回使った魔改造webpack.configを貼っておきます。

https://gist.github.com/yuroyoro/221224204f65d8e3fddaf293e5571265

