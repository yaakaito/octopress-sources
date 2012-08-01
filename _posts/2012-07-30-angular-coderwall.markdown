---
layout: post
title: "Angular.jsでcoderwallのバッチ取得してみる"
date: 2012-07-30 00:53
comments: true
categories: ["JavaScript", "AngularJS"]
---

## Coderwallのバッチ取得するのはよくみるけど
[Angular](http://angularjs.org/)でやってみた的なのはみないなーと思ったのでやってみた。
といってもjQueryなんかでやるのとそんなに変わらないです。

## バッチを表示させるHTMLを書く
とりあえずリストにイメージを突っ込んでいく感じにすることにしました。
```html
<div ng-controller="coderwallController">
  <ul>
    <li ng-repeat="badge in badges">
      <img ng-src="{% raw %}{{badge.badge}}{% endraw %}" alt="{% raw %}{{badge.description}}{% endraw %}" title="{% raw %}{{badge.name}}{% endraw %}" />
    </li>
  </ul>
</div>
```
こんな感じでデータをバインディングします。

## データを取得する
jQueryなんかでgetJSONとかするのとそんなに変わらないです。
```javascript
function coderwallController($scope, $http) {

  $scope.badges = [];   
  $http.jsonp("http://coderwall.com/yaakaito.json?callback=JSON_CALLBACK").success(function(data,status,header,config){
    $scope.badges = data.data.badges;
    console.log($scope.badges);
  });
}
```
で、取得できました！やりましたね！


全文はgistにあります。

{% gist 3199870 %}
