---
layout: post
title: "Octopressのデザインを楽に開発するmakefile"
date: 2012-07-22 13:09
comments: true
categories: 
---

## こんにちは！
この記事は僕がプレビュー用にも使っているので無駄な部分も多いですが、あまり気にしないでくださいね！

## テンプレートをいじろう
octopress/.themeの中に適当な名前でディレクトリを作って
```
$ rake install\["hoge"\]
```
とかすればテンプレートがインストールできるわけですが、これをしたあとに直接soruceを弄って〜みたいなことは嫌な訳です。
ですが僕は天才なのでmakeを使うことにしました。
```
all:
  cd octopress
  yes | rake install\["hoge"\]
  rake generate
  rake preview
```
これで.theme/hogeの方からmakeをするだけで、.theme/hogeの中で修正したものをインストールしてコンパイルしてくれるようになりました、やりましたね！！！

## 今日も天気がいいですね
天気がいいとObjective-Cが書きたくなりますよね。
```objective-c
[NSString stringWithFormat:@"スティーブ・天気がいい・ジョブズ];
```
JavaScriptもいいかもしれません。
```javascript
var jobs = "スティーブ・天気がいい・ジョブズ";
```

## こういうの
こういうのでるかな ``hoge`` ？

## 明日は晴れるでしょうか
晴れるといいですね。
{% img http://cdn-ak.f.st-hatena.com/images/fotolife/y/yaakaito/20120117/20120117065430.png %}
