---
layout: post
title: "ParseとAngularJSでユーザー毎にデータ同期してみる"
date: 2012-08-08 01:31
comments: true
categories: ["Parse", "AngularJS", "JavaScript", "WebApp", "Backbone.js"]
---

こんにちは！
最近[Parse](http://parse.com)というバックエンドを肩代わりしてくれるサービスを知ったので、
ちょっとAngularJSと組み合わせてユーザー毎の情報を同期する程度のアプリを作ってみました。
どうやら小規模なサービスでユーザー情報の同期とかに使うといいよ、みたいな感じらしいので、
利用用途としても今作っているものにあっていそうだったので、とりあえずテストで作ってみた感じです。

## Parseに登録する
[Parse](http://parse.com)にいって、「Try it for free」します。
無料版だとAPI利用回数制限とかが結構厳しそうに見えますが、個人で作るくらいなら全然余裕なくらいはありますね。
ログインするとDashboradにいけるので適当な名前で新しいアプリを作ります。
QuickStart的な画面になると思うので、JavaScript用のSDKで、「New Project」を選択します。
そうすると、ここからSDKをダウンロードしてね！という画面が下の方に出ているので、SDKをダウンロードしましょう。
SDKをダウンロードしたら、このキーを使ってね、というのが出ているはずなので、`index.html`を弄ってキーを書き換えて、開いてみましょう。
失敗すると「おい違うぞ」みたいな分かりやすい感じになるので、そうならなければ成功です。ついでにQuickStartにある「Test the SDK」も試してみましょう。
これでとりあえずセットアップは終わりです。

## チュートリアルを見ながらちょっと書いてみよう
[JavaScriptのチュートリアル](https://www.parse.com/tutorials/todo-app-with-javascript)をみてみましょう。
ありがたいことに、まさにやりたいことが書いてありそうなチュートリアル準備されているので、とりあえずこれをみてみます。
[JavaScriptのガイド](https://www.parse.com/docs/js_guide)も一緒にみると良さげなので、これも見てみることにします。
どうもSDKは[Backbonejs](http://documentcloud.github.com/backbone/)ベースみたいです。正確にはbackboneライクという感じですが。
これってjQueryとprototype同居の悪夢じゃね的な感じがありますね。わかんないですけど。どうしましょうかね。
普通に[REST API](https://parse.com/docs/rest)もあるので素直にAngularResouce使った方がいいんじゃないですかね。感出てきました。

まあ今回はどんなものか試してみるだけのものなので失敗込みで、ともかく進めていきましょう。
方針としてはbackboneライクのモデルだけ使ってその他はAngularJSに持ってもらうイメージでいきます。
チュートリアルはどうやら[TODOMVC](http://addyosmani.github.com/todomvc/)のbackbonejs版を拡張しているみたいです。
[コードはGithub](https://github.com/ParsePlatform/Todo)にあるようです。テンプレートとかは放っておいてコアっぽいところを見ていきます。
まずログインしてるかしてないか、みたいなところは`Parse.User.current()`で判定できるみたいです。
```javascript
if (Parse.User.current()) {
  new ManageTodosView();
} else {
  new LogInView();
}
```
で、ログインしてなかったらLogInView表示してユーザー作れよってことっぽい。
ユーザー作るあたりを見てみると、
```javascript
Parse.User.signUp(username, password, { ACL: new Parse.ACL() }, {
  success: function(user) {
    new ManageTodosView();
    self.undelegateEvents();
    delete self;
  },
  error: function(user, error) {
    self.$(".signup-form .error").html(error.message).show();
    this.$(".signup-form button").removeAttr("disabled");
  }
});
```
ほほう、ACLってどういうオプションだろ、と見てみると、
```
This creates the account with the given username and password from the fields and also applies a blank ACL on the user. This prevents anyone from reading data from the User class unless they are the user who is logged in.
```
と書いてあるので、このユーザーから登録したデータはほかのユーザーからは取得できないものだよ、ということですかね。
つづいてログインはこんなん
```javascript
Parse.User.logIn(username, password, {
  success: function(user) {
    new ManageTodosView();
    self.undelegateEvents();
    delete self;
  },
  error: function(user, error) {
    self.$(".login-form .error").html("Invalid username or password. Please try again.").show();
    this.$(".login-form button").removeAttr("disabled");
  }
});
```
ほほう、で、モデルの管理はBackboneライクなわけですね！やりましたね！
```javascript
var Todo = Parse.Object.extend("Todo", {
  // ...
});
var TodoList = Parse.Collection.extend({
  // ...
});
```
こんな感じでbackboneライクにモデルを作って`save`を呼ぶとサーバーへ遅れる感じですね。
取得は
```
// Create our collection of Todos
this.todos = new TodoList;

// Setup the query for the collection to look for todos from the current user
this.todos.query = new Parse.Query(Todo);
this.todos.query.equalTo("user", Parse.User.current());

// ...

// Fetch all the todo items for this user from Parse
this.todos.fetch();
```
という感じっぽいです。

## Angularで書いてみよう
使い方も分かってきたのでAngularで書いてみましょう。どきどきですね。＞＜
コントローラーやイベントハンドリングとかの基本的な部分はすべてAngularで面倒をみるので、
まずは普通にそれっぽいビューを作っていきます。ライブラリを読み込んで、
```javascript
    <script type="text/javascript" src="http://code.angularjs.org/angular-1.0.1.min.js"></script>
    <script type="text/javascript" src="http://www.parsecdn.com/js/parse-1.0.14.min.js"></script>
    <script type="text/javascript" src="./javascripts/app.js"></script>
```
適当な感じにビューを作ります。
```html
<div ng-controller="AppCtrl">
  <section>
    <h2>account</h2>
    <div>
      <input ng-model="naccount" type="text" />
      <input ng-model="npass" type="password" />
      <button ng-click="createAccount()">create</button>
    </div>
    <div>
      <input ng-model="laccount" type="text" />
      <input ng-model="lpass" type="password" />
      <button ng-click="login()">login</button>
    </div>
  </section>
  <section>
    <h2>Objects</h2>
    <button ng-click="sync()">sync</button>
    <div>
      <input ng-model="itemName" type="text" />
      <input ng-model="itemDescription" type="text" />
      <button ng-click="addItem()">add</button>
    </div>
    <ul>
      <li ng-repeat="item in items">
        {{ item.name }} / {{ item.description }}
      </li>
    </ul>
  </section>
</div>
```
コントローラーはとりあえずこんな感じで。
```javascript
function AppCtrl($scope) {
  $scope.createAccount = function() {
  
  };

  $scope.login = function() {

  };

  $scope.addItem = function() {
    
  };
}
```

準備が整ったので、まずはParseを初期化します。
コントローラーが読み込まれたあとに、
```javascript
Parse.initialize("hoge", "hoge");
```
とします。そうしたらモデルとそのコレクションを定義します、こんな感じかな。
```javascript
var SyncAppObject = Parse.Object.extend("SyncAppObject", {
  default: {
    description: "description";
  }
});
var SyncAppObjectList = Parse.Collection.extend({
  model: SyncAppObject
  , comparator : function(obj) {
    return obj.get("name");
  }
});
  
```

モデルの準備ができたので、addItemでオブジェクトを作ってみましょう。
```javascript
var list = new SyncAppObjectList();
$scope.items = list.models;

$scope.addItem = function() {
  var item = new SyncAppObject({name : $scope.itemName, description : $scope.itemDescription});
  list.add(item);    
};
```
コレクションのインスタンスを作って、`$scope.items`へ`models`を関連づけます。
あとは`addItem`で新しいオブジェクトを作って追加してあげるだけです。
が、Backbone的にはプロパティへのアクセスは`get()`を使ってね、ということなので、テンプレートの方も少し修正します。
```
      <li ng-repeat="item in items">
        {{ item.get("name") }} / {{ item.get("description") }}
      </li>
```
これで、addを押すと、BackboneライクなオブジェクトをAngularで表示できているはずです。予想に反して問題なさそうですね。
次にこのモデルをParseへ送りつけましょう。`save`を呼ぶだけでokです。
```
  item.save();
```
うまくいけばParseのアプリケーションのマネージメニューから、「Data Browse」をすると、ちゃんとデータが追加されているはずです。



## 結論
案外うまくいった。

## 教訓
ずっとスティーブ・Objective-C・ジョブズしててJavaScript界隈についていけていないので遅れ取り戻さないとやばいですねー。
