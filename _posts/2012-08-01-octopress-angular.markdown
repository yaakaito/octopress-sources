---
layout: post
title: "OctopressでAngularJSを使う"
date: 2012-08-01 02:54
comments: true
categories: ["AngularJS", "JavaScript", "Octopress"]
---

こんにちは！[Octopress](http://octopress.org)、カスタマイズしてますか？
えっ、テンプレートを変えただけ？？？自分で書きましょう！！！！

## AngularJS使えば夢が広がるんじゃね？
というわけでOctopressで[AngularJS](http://angularjs.org)使ってみよう的な話です。
まあ別にjQueryでもいいんですけど、Octopressは静的ページですし、
検索なんかもデフォルトはgoogleに飛ぶだけとか、結構貧相じゃないですか。
結局のところ単純に使いたかったのが大きいんですが、
データさえ埋め込んでおけば楽にいろいろ作れる感じがしたので、AngularJS使ってみることにしました。

## そもそもシンタックスがぶつかるんですけど
[Handlebars.js](http://handlebarsjs.com/)なんかでもそうですが `{% raw %}{{ hoge }}{% endraw %}` というシンタックスがぶつかりますね。
これは[Jekyll](http://jekyllrb.com/)のプラグインにrawプラグインというのがあるのでそれを使うと解決できます。
ただ、よくみたらOctopressには最初からrawプラグインがついていたので、ありがたくこれを使わせてもらいましょう。
AngularJSの対象になるところは、(ネストできなくて外側でお茶を濁してるので\は外してくださいね)
```
{\% raw %}
  {% raw %}{{ hoge.fuga }}{% endraw %}
{\% endraw %}
```
とか書けばよいです。簡単ですね。

## AngularJSを読みこむ
問題も解決できましたし、とりあえずAngularJSを読み込みましょうか。
`source/custom/head.html` をいじります。
適当な場所で
```html
<script src="http://code.angularjs.org/angular-1.0.1.min.js"></script>
```
とかいて、AngularJSを読み込みます、あとはhtmlにng-appを付けましょう。
```html
<!--[if IEMobile 7 ]><html class="no-js iem7" ng-app><![endif]-->
<!--[if lt IE 9]><html class="no-js lte-ie8" ng-app><![endif]-->
<!--[if (gt IE 8)|(gt IEMobile 7)|!(IEMobile)|!(IE)]><!--><html class="no-js" lang="en" ng-app><!--<![endif]-->
```
これでAngularJSを使う準備ができました！めっちゃ簡単ですね！: )

## Githubのリポジトリを取ってみる
今作っているテンプレート(このテンプレートですね)では下の方にgithubのリポジトリが何個かでるので、これを作ってみましょう 。`_include/asides/github.html` をこんな感じにします。
```html
{% raw %}<div ng-controller="githubCtlr">
  <ul class="repositories">
    {\% raw %}
      <li ng-repeat="repo in repos" data-lang="{{ repo.language | lowercase }}">
        <a href="{{ repo.html_url }}">{{ repo.full_name }}</a>
      </li>
    {\% endraw %}
  </ul>
</div>{% endraw %}
```
これにあわせたJSを書きます。
```javascript
<script>
  function githubCtlr($scope, $http) {
    $scope.repos = [];
    $http.jsonp("https://api.github.com/users/{% raw %}{{ site.github_user }}{% endraw %}/repos?page=1&per_page=20&sort=pushed&callback=JSON_CALLBACK").success(function(data,status,header,config){
      var count = {\{ site.github_repo_count }} || 5;
      for(var i = 0; i < count; i++) {
        $scope.repos.push(data.data[i]);
      }
    });
  }
</script>
```
`_config.yml` はちゃんと生かしたいのでちょっと面倒くさい感じになってますね。
まあ20件くらい取ってこれば大丈夫やろ、的なザツい感じになってます。
ともかくこれでリポジトリを取ってくることができました！やりましたね！

デモはページの下の方をみてください！


## エントリ本文もAngularされちゃう！
で、ここまではいいんです。
けどAngularJS弄ってみたりしたらまあそれをブログに書くのは当然だよねーみたいなところで少し困ったことになります。
なんとブログ本文もAngularJSの実行対象になってしまいました！やりましたね！！！

raw使って生の `{% raw %}{{ hoge }}{% endraw %}` にしても、
Angularでその表示はなかったことにされるので、何が起こったのか一瞬分からなくなります。
これは困るので解決しましょう、AngularJSを使う場所をそもそも絞ってしまうのがよいですが、
面倒なので `ng-non-bindable` を使いましょう。 `ng-non-bindable` で囲まれた中はAngularJSの対象外になるので、何が起こったのか分かるようになります。

`_include/article.html` なんかで `{% raw %}{{ content }}{% endraw %}` の周りをガッツリ囲んでしまうのが楽です。
```html
<section><div ng-non-bindable>{% raw %}{{ content }}{% endraw %}</div></section>
```
これでブログが書けるようになりました！やりましたね！
Octopress自体のことを書くときも同じことなので気をつけましょう。

デモはこのエントリです。


## できた！
これで夢が広がりましたね！！！！


## 全然関係ないですが
githubAPIでリポジトリ引いてくるみたいなコードよく見ますけど、
クエリーなしだと30件しか帰ってこなくて全部みれないし、ソートが単なる名前順で微妙なので、
`page` + `per_page` と `sort` くらいは指定した方がいいと思いますよ。

