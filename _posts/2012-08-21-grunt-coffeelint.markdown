---
layout: post
title: "gruntのcoffeelintタスク作ってみた"
date: 2012-08-21 01:59
comments: true
categories: [JavaScript, CoffeeScript, Node.js, grunt]
---

こんにちは！うきょーです！
みなさん[CoffeeLint](http://www.coffeelint.org/)使ってますか？
僕はあんまりCoffeeは書かないんですが、ちょっと使ってみようかなーと思っているアプリがあるので、それの下準備をしています。
coffeelintは便利ですがいちいち
```
$ coffeelint hoge.coffee
```
とかするのはだるいですよね！
なので[grunt.js](https://github.com/cowboy/grunt)を使いましょう！
(grunt.jsの説明は別にしません)

## 使い方

ほぼ自分用でnpmとかには登録してないのでがんばってください！

* [grunt-coffeelint](https://github.com/yaakaito/grunt-coffeelint)

まずはCoffeelintを入れます。
```
$ npm install -g coffeelint
```

取ってきます。
```
$ git clone git://github.com/yaakaito/grunt-coffeelint.git grunt-coffeelint
```

タスクをコピーします。
```
cd grunt-coffeelint
cp -rf tasks your/grunt/dir
```

あとはロードしてconfigを埋めます。
```javascript
grunt.loadTasks('tasks');
grunt.initConfig({
  // ...
  coffeelint : {
    all : { 
      files : [
        'coffee/*.coffee'
      ]
    }
  }
});
```

いざ！
```
$ grunt coffeelint
Running "coffeelint:all" (coffeelint) task
[ 'coffee/*.coffee' ]

  ✓ coffee/a.coffee
  ✓ coffee/b.coffee

  ✓ Ok! » 0 errors and 0 warnings in 2 files


Done, without errors.
```

やりましたね！！！！

## まとめ
grunt.js便利なので使いましょう。

