---
title: EC2インスタンス料金月額日本円化
tags: AWS
author: yuroyoro
slide: false
---
[EC2 インスタンスの料金 – アマゾン ウェブ サービス \(AWS\)](https://aws.amazon.com/jp/ec2/pricing/on-demand/) で、


デベロッパーズコンソール開いておもむろにこれを実行すると

```js
$("td.rate").each(function(i,e){var s=$(e).text(); var v=s.match(/\$([0-9\.]+)/)[1]*24*30*112;$(e).text(s+" | 月額 "+v.toFixed(0)+"円");});
```

こうじゃ(112 JPY/USD)

<img width="857" alt="Screen Shot 2017-06-28 17.54.34.png" src="https://qiita-image-store.s3.amazonaws.com/0/64/4b3ca79f-b516-7287-9086-86e0b6dc1731.png">

ブックマークレットとしてブクマしておくのもよい

```js
javascript:$("td.rate").each(function(i,e){var s=$(e).text(); var v=s.match(/\$([0-9\.]+)/)[1]*24*30*112;$(e).text(s+" | 月額 "+v.toFixed(0)+"円");});
```



