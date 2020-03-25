---
title: CapybaraでUnicornを使う
tags: Ruby Capybara
author: yuroyoro
slide: false
---
Capybaraはデフォルトでwebrickが動くようになっているが、`Capybara.server` に他のRackサーバーを起動するProcを渡すことで変更することが可能。以下はUnicornでCapybaraする例。

```rb
  Capybara.server do |app, port|
    Unicorn::Configurator::RACKUP[:port] = port
    Unicorn::Configurator::RACKUP[:set_listener] = true

    # portやworker数は好みで変えてな
    server = Unicorn::HttpServer.new(app, worker_processes: 2)

    # Unicornを起動した場合、Capybaraが動いているProcessが
    # Unicornのmasterとなり、workerがforkされる
    # spec終了時にUnicornを停止させるようtrapしておく
    at_exit do
      server.stop(false)
    end

    server.start

    # Capybara2.0では、Capybara.serverに渡すblockは
    # ブロッキングすることが期待されているのでThreadを止める
    Thread.stop
  end
```

