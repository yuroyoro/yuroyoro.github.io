---
title: Goで外部コマンドをパイプして実行する
tags: Go
author: yuroyoro
slide: false
---
Golangで、`git log --oneline | wc -l` のように複数コマンドをパイプして実行結果を取得したい。

以下のように、複数の`Cmd` を用意して標準出力と標準入力を繋いでやればいい。

```go
	c1 := exec.Command("git", "log", "--oneline")
	c2 := exec.Command("wc", "-l")

	r, w := io.Pipe()
	c1.Stdout = w
	c2.Stdin = r

	var out bytes.Buffer
	c2.Stdout = &out

	c1.Start()
	c2.Start()
	c1.Wait()
	w.Close()
	c2.Wait()

        fmt.Println(string(out))
```

正直ダルいので、雑にやるならこう

```go
	c, _ := exec.Command("sh", "-c", "git log --oneline | wc -l").Output()
	fmt.Println(string(out))
```

`sh` 経由にしてしまう。環境変数とか色々と面倒なことが起こりえるので注意。

もっとうまいやり方誰か教えてください( ꒪⌓꒪) 

