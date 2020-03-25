---
title: Goでjqっぽくinterface{}の中を探索する
tags: Go
author: yuroyoro
slide: false
---
Goで、 `json.Unmarshal` の結果を `interface{}` 型で受け取ると、中を探索するのにキャストが発生してダルい。

```go
  var obj interface{}
  raw := `{"foo": [{"bar":1}, {"baz": 2}]}`
  json.Unmarshal([]byte(raw), &obj)
  tmp := obj.(map[string]interface{})
  fmt.Printf("%+v\n", tmp["foo"])
  foo := tmp["foo"].([]interface{})
   

  fmt.Println(foo[0].(map[string]interface{})["bar"])
```
https://play.golang.org/p/3NX4W6ZVCw

そこで、[dockerのコード](https://github.com/docker/docker/blob/master/utils/templates/templates.go)を参考に、 `text/template` を使って`inetrface{}`の中を探索するコードを書いてみた

```go
package main

import (
	"fmt"
	"bytes"
	"encoding/json"
	"text/template"
)

func main() {
  raw := `
{ "foo" :[
   { 
     "bar1" : { "hoge" : "aaaa"},
     "bar2" : { "hoge" : "bbbb"}
    },
    {
     "baz1" : { "fuga" : "cccc"},
     "baz2" : { "fuga" : "dddd"}
    }
  ],
  "noooo" : [ 33,44,55]
}`
  var obj interface{}
  err :=json.Unmarshal([]byte(raw), &obj)
  if err != nil {
    panic(err)
   }

   queries := []string{
     "{{ .foo }}",
     "{{ index .foo 0 }}", 
     // 配列の中を探すときは {{ with index ...}} {{ ... }}{{ end }} にしなければならぬ
     "{{ with index .foo 0 }}{{ .bar1 }}{{ end }}", 
     "{{ with index .foo 0 }}{{ .bar1.hoge }}{{ end }}",
     "{{ with index .foo 1 }}{{ .baz2.fuga }}{{ end }}",
     "{{ index .noooo 0 }}",
     "{{ index .noooo 1 }}",
    }

   for _, q := range queries {
     fmt.Printf("query : %s -> %s\n", q, query(obj, q))
   }

}

func query(obj interface{}, q string) string {
  t := template.Must(template.New("query").Parse(q))
  var out bytes.Buffer
  t.Execute(&out ,obj)

  return out.String()
}
```
https://play.golang.org/p/OxpnDGSR4a

`text/template` を使うので、結果は全て `string`型になる。

また、配列が挟まると{{ with index .key 0 }}{{ .hoge }} {{ end }}のようにwithを使わないとならないが、いちいちinterface{}からmapやarrayにcastしなくてもよいのでパフォーマンス気にしなければそれなりに便利っぽい?

ognl式でstructを探索できるような、もっとよいライブラリがあったら教えてくださいᕕ( ᐛ )ᕗ

