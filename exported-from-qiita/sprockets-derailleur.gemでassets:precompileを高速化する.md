---
title: sprockets-derailleur.gemでassets:precompileを高速化する
tags: Ruby Rails
author: yuroyoro
slide: false
---
ぼくのassets:precompileへの熱い想いは [こちら](https://twitter.com/search?q=assets%3Aprecompile%20from%3Ayuroyoro&src=typd&lang=ja) をご確認ください。

で、あんまりにも遅いので調べていたらこんな [こんなIssue](https://github.com/sstephenson/sprockets/issues/308) を発見した。
なんかforkしてcore使うようにして高速化してくれるgem作ったぜボーイズ! みたいなこと書いてある！ これは……!!

早速試す。

[steel/sprockets-derailleur](https://github.com/steel/sprockets-derailleur)

# setup

Rails4系なら、Gemfileに`gem 'sprockets-derailleur'`を追加して、`bundle install` する。
で、`config/environmet.rb` とかに`require 'sprockets-derailleur'` と書いておく。

Rails3.2系なら、上記に加えて `config/initializers/sprockets_derailleur.rb` というファイルを以下の内容で作成する

```ruby
module Sprockets
  class StaticCompiler

    alias_method :compile_without_manifest, :compile
    def compile
      puts "Multithreading on " + SprocketsDerailleur.worker_count.to_s + " processors"
      puts "Starting Asset Compile: " + Time.now.getutc.to_s

      # Then initialize the manifest with the workers you just determined
      manifest = Sprockets::Manifest.new(env, target)
      manifest.compile paths

      puts "Finished Asset Compile: " + Time.now.getutc.to_s

    end
  end

  class Railtie < ::Rails::Railtie
    config.after_initialize do |app|

      config = app.config
      next unless config.assets.enabled

      if config.assets.manifest
        path = File.join(config.assets.manifest, "manifest.json")
      else
        path = File.join(Rails.public_path, config.assets.prefix, "manifest.json")
      end

      if File.exist?(path)
        manifest = Sprockets::Manifest.new(app, path)
        config.assets.digests = manifest.assets
      end

    end
  end

end
```


注意点として、`config/initializers`にhookを入れるので、`application.rb`とかで`config.assets.initialize_on_precompile = false` に設定してると動作しない。
その場合は、同オプションをコメントアウトして対処する(その副作用として、assets:precompile時にDBへ接続しにいく、などが起こりうるが……)

# どのくらい速くなったか

手元の8core, memory 16Gのマシンで、導入前後の実行時間。

before 

```

$ time rake assets:precompile
noglob rake assets:precompile  620.77s user 9.48s system 100% cpu 10:26.73 total


```


after

```
$ time rake assets:precompile
noglob rake assets:precompile  1847.59s user 143.09s system 511% cpu 6:28.89 total
```

きっちりCPUコア使ってますね。
10分26秒から6分28秒に高速化。
これは、なかなかよいのではないでしょうか？！！！！？？？

