---
layout: post
title: "便利なライブラリHullToRefresh作った"
date: 2012-08-31 06:31
comments: true
categories: [Objective-C, iOS, Joke]
---

こんにちは！うきょーです！みなさんPullToRefresh使ってますか？便利ですよね。
けど、引っ張って更新、そろそろ飽きてきませんか？てゆーか、なんでわざわざ引っ張らなきゃいけないんですか？

それを解決するために、HullToRefreshというライブラリを作りました。名前はギャグっぽいですがマジメです。

* [HullToRefresh](https://github.com/yaakaito/HullToRefresh)

振るとRefreshはその名の通り、iPhoneを大きく振るとイベントが飛んでくるライブラリです、すごく、すごく便利ですね。
なんてったって画面に触る必要がありません、指を動かす必要もありません。更新したいな、と思ったときにはiPhoneでフリフリシェイクすればいいのです、楽しいですね。
電車に乗っているときは自動で揺れを検知し、空気を読んで更新をしてくれます。なんて便利なんでしょう、指による入力なんて、もはや時代遅れなのです。
このすばらしいアイデアの使い方はすごく簡単で、initしてフルフルNotificationを登録するだけです！！！

```objective-c
[HullToRefresh sharedHullHull];
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(callback)
                                             name:kDidHullHullNotification
                                           object:nil];
```

ね？簡単でしょう？サンプルについてくるアプリはこんな感じになります。

{% img /images/hullhull.png %}

すばらしいライブラリなので、是非使ってみてください！使いどころとしては、空気を読まないアプリケーション作るときに便利です。


### まとめ

完全にギャグです。タイポしたときとかに出るように割と悪意を持って作っていますが、ただのギャクです。
