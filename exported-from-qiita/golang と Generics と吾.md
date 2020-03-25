---
title: golang と Generics と吾
tags: Go
author: yuroyoro
slide: false
---
吾はGoでGenericsがないことに関してはわりと肯定的な立場ではあるのだが、流石に「[golang と Generics と私](http://mattn.kaoriya.net/software/lang/go/20170309201506.htm) 」の記事の例はどうかと思ったので、畳み込みfold関数を例にGenericsが解決する問題を例示してみようと思う。

なぜfoldかというと、 `List<T>` の要素を加算して集約する処理を書くなら普通はfoldで実装するし、foldがあればmapもfilterも実装できるので。

## javaで畳み込み

Stream APIで用意されてるreduceで一発です

```java
    List<Integer> list = Arrays.asList(1, 2, 3);

    // listの加算とか畳み込みで一発ですよ
    int result = list.stream().reduce((a, b) -> a + b).get();
```

https://gist.github.com/yuroyoro/e0cd9861173df3ee1a19b9e9f44355fd#file-sum-java

それだと話が終わってしまうので、 このfold関数を自作してみる

```java
    // 自作fold関数
    // 引数に List<T> と、2引数の演算を行うBiFunction<T, T, T>をとって、
    // listを順番にfで畳み込む
    public static <T> T fold(List<T> list, BiFunction<T, T, T> f) {
        T res = list.get(0);
        for(Iterator<T> it = list.listIterator(1); it.hasNext(); ){
            res = f.apply(res, it.next());
        }
        return res;
    }
```

このfold関数は型Tで抽象化されており、Tが`Integer`だろうが`String`だろうが対応する`BiFunction`を渡すことでどんな型のコンテナにも畳み込みができる。

以下のように、 `List<Interger>` に対しては要素の合算をfoldで計算し、 `List<String>` では文字列を","で結合するために利用可能である。

```java
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

    // 自作fold関数でも同じ
    int result = fold(list, (a, b) -> a + b );

    System.out.println(result);

    List<String> slist = Arrays.asList("foo", "bar", "baz");

    // 文字列を, で結合する関数をfoldに渡す
    String str = fold(slist, (a, b) -> a + ", " + b);

    System.out.println(str);
```

https://gist.github.com/yuroyoro/e0cd9861173df3ee1a19b9e9f44355fd#file-fold-java

このように、型Tで抽象化されたコンテナに対して、そのコンテナの畳み込み操作と、型Tに対する具体な処理を行う関数を引数にもらって分離することで、具体的な型Tが何かを知らなくても畳み込みという操作を抽象化している。

この要領で、mapやfilterなど様々なコンテナ操作を抽象化することができる。

このコンテナ操作を抽象化する、という機能は、別にGenericsがなくても実装は可能である。しかし、これを型安全に行うためにはGenericsの支援が必要なのである。

## golangで畳み込み

ではgolangでfold関数を実装してみよう。

```go
// reflectで気合でfoldするやつ
// もちろん型不安全
func fold(l interface{}, f interface{}) interface{} {

	lv := reflect.ValueOf(l)
	fv := reflect.ValueOf(f)

	size := lv.Len()
	v := lv.Index(0)
	for i := 1; i < size; i++ {
		v = fv.Call([]reflect.Value{v, lv.Index(i)})[0]
	}

	return v.Interface()
}
```

fold関数のシグニチャを見てもらえばわかると思うが、 `interface{}` ばかり出てくる。 javaで考えると引数がすべて `Object` だと思えばいい。

```go
	list := []int{1, 2, 3, 4, 5}
	res := fold(list, func(a int, b int) int { return a + b }) // 戻り値はinterfaceになる
	fmt.Println(res)

	slist := []string{"foo", "bar", "baz"}
	sres := fold(slist, func(a string, b string) string { return a + ", " + b })
	fmt.Println(sres)

	// string型のlistにint型を取る関数を渡すと実行時エラー
	sres = fold(slist, func(a int, b int) int { return a + b })
	fmt.Println(sres)

```

いちおう使えることは使えるが、 fold関数の戻り値は `interface{}` 型なので、intとして使うには型アサーションが必要だし、コンテナ型と合わない型の関数を渡すと実行時エラーになる。Genericsがあればこの手の実行時型エラーはコンパイル時に潰すことができる

## つまり

GoにGenericsがほしいっていう人の9割5分はmap,filter,foldなどの抽象化された型安全なコンテナ操作がほしいのであって、この問題が解決できないにもかかわらず、GoはGenericsなくても大抵の問題は解決できるし使う頻度も少ない、と言っても納得しないだろう。

のこりの5分の人はありとあらゆる言語でモナドを実装しないと気がすまないタイプだと思うのでそういう人たちは無視していい。

で、goにはこの手の抽象化されたコンテナ関数はなく、sliceの操作は気合と筋力でforを書いていくスタイルになるのでつまり

<blockquote class="twitter-tweet" data-lang="en"><p lang="ja" dir="ltr">Genericsないなら筋力をつけろ</p>&mdash; null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/839313607615655936">March 8, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" data-lang="en"><p lang="ja" dir="ltr">genericsないので抽象化されたコンテナ型とその操作関数が提供されておらず筋力でfor文を書いていくんだよつまり必要なのは筋力</p>&mdash; null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/839313539990900736">March 8, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

