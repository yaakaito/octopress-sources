---
layout: post
title: "ParseとAngularJSでユーザー毎にデータ同期してみる"
date: 2012-08-09 02:29
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
チュートリアルのコードを追っているのでチュートリアルからの転載です。
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
### とりあえず簡単なビューをつくる
コントローラーやイベントハンドリングとかの基本的な部分はすべてAngularで面倒をみるので、
まずは普通にそれっぽいビューを作っていきます。ライブラリを読み込んで、
```javascript
    <script type="text/javascript" src="http://code.angularjs.org/angular-1.0.1.min.js"></script>
    <script type="text/javascript" src="http://www.parsecdn.com/js/parse-1.0.14.min.js"></script>
    <script type="text/javascript" src="./javascripts/app.js"></script>
```
{% raw %}
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
    <button ng-click="syncItems()">sync</button>
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

  $scope.syncItems = function() {

  }

}
```

### Parseを使ってみる

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
var objectList = new SyncAppObjectList();
$scope.items = objectList.models;

$scope.addItem = function() {
  var item = new SyncAppObject({ name : $scope.itemName
                                 , description : $scope.itemDescription});
  objectList.add(item);
  $scope.items = objectList.models;
};
```
コレクションのインスタンスを作って、`$scope.items`へ`models`を関連づけます。
あとは`addItem`で新しいオブジェクトを作って追加してあげるだけです。
が、Backbone的にはプロパティへのアクセスは`get()`を使ってね、ということなので、テンプレートの方も少し修正します。
```javascript
      <li ng-repeat="item in items">
        {{ item.get("name") }} / {{ item.get("description") }}
      </li>
```
これで、addを押すと、BackboneライクなオブジェクトをAngularで表示できているはずです。予想に反して問題なさそうですね。
次にこのモデルをParseへ送りつけましょう。`save`を呼ぶだけでokです。
```javascript
item.save();
```
うまくいけばParseのアプリケーションのマネージメニューから、「Data Browse」をすると、ちゃんとデータが追加されているはずです。

### ユーザー作成とログイン
まずはユーザーを作りますしょう。`createAccount`でチュートリアルでもでてきたsignupするコードを書きます。
```
$scope.createAccount = function() {
  Parse.User.signUp($scope.naccount
                    , $scope.npass
                    , { ACL: new Parse.ACL() }
                    ,  
  {
    success: function(user) {
      alert("ユーザー登録に成功したよ。 o(*^▽^*)o");
    }
    , error: function(user, error) {
      alert("ユーザー登録に失敗しちゃったよ。 (ﾉ_･｡)");
    }
  });
};
```
フォームからアカウントを作ってみましょう。成功したメッセージが出たら、「Data Browse」から確認してみましょう。
Userというテーブルが増えているはずです。やりましね！
ユーザーを作ったらそれに関連するデータとして登録するようにしましょう。
ユーザーに関連づけるには`user`と`ACL`を設定します。
```javascript
$scope.addItem = function() {
  if(!Parse.User.current()) {
    alert("ログインするかユーザーつくってね。ヾ(@~▽~@)ノ");
    return ;
  }
  var item = new SyncAppObject({name : $scope.itemName
                                , description : $scope.itemDescription
                                , user : Parse.User.current()
                                , ACL : new Parse.ACL(Parse.User.current())});
  objectList.add(item);
  item.save();   
  $scope.items = objectList.models;
};
```
これでデータを追加してみて、「Data Browse」から確認します。するとユーザーのobjectIdと関連づけられているはずです。
カラムも自動で拡張されるみたいです。便利ですね。
ところでここで作ったユーザーとか後でやるログインした状態とかはSDK側でローカルへキャッシュしてくれるみたいです。
だいたいできてきました！(だんだん顔文字がうざくなってきましたね)次はログインを作りましょう。
```
$scope.login = function() {
  Parse.User.logIn($scope.laccount, $scope.lpass, {
    success: function(user) {
      alert("ログインに成功したよ。 o(*^▽^*)o");
    },
    error: function(user, error) {
      alert("ログインに失敗しちゃったよ。 (ﾉ_･｡)");
    }
  });
};
```
こんな感じにして、作ったユーザーでログインできるか試してみましょう。
ログインできたらユーザーを切り替えたりしてみて、「Data Browse」から確認してみましょう！どうですか？成功しましたか？やりましたね！
ついでなのでユーザー名を表示するようにしておきましょう。
```html
 <p>Login User : {{ loginUser }}</p>
```
こんな感じにかいて、初期化時に、
```javascript
if(Parse.User.current()) {
  $scope.loginUser = Parse.User.current().get("username");
} else {
  $scope.loginUser = "not login";
}
```
とかして初期の表示を作ってあげて、さらにユーザー作成やログインにフックしてビューを更新してあげます。
```javascript
  success: function(user) {
    $scope.loginUser = Parse.User.current().get("username");
    $scope.$apply();
    alert("ログインに成功したよ。 o(*^▽^*)o");
  },
```
### 同期を実装する
コレクションに対して`fetch`することで取得できるはずだったので、マージとか考えなければ案外楽にいけそうです。
今回はマージとかは何も考えてなくて、とにかくサーバーからデータを拾ってくるだけです。
条件なんかをちょこちょこ書いてビューへ反映しましょう。
```javascript
$scope.syncItems = function() {
  objectList.query = new Parse.Query(SyncAppObject);
  objectList.query.equalTo("user", Parse.User.current());
  objectList.fetch();
  $scope.items = objectList.models; 
}
```
これでsyncを押せばサーバーからログインしているユーザーにあわせたデータを拾ってこれま・・・したが、
AngularJSのビューの更新との相性が悪いのか、1回目のsyncだとビューに反映されないですねっていう。
このへんはちゃんと調べてないので分かんないですが、あんまりよくないですねー。が、今回はParseがどんなもんか試すだけなのでまあおけおけ。
とりあえずこんな感じで実装はおしまいです。

{% endraw %}

## コードとか
コードはgithubにあります。

[yaakaito/sync-app](https://github.com/yaakaito/sync-app)

またサンプルをgh-pagesへデプロイしてあるので、どんな感じかなーと気になる人がいれば見てみるとなにかよいかもしれません。

[yaakaito.github.com/sync-app/](http://yaakaito.github.com/sync-app/)

関係ないですが、簡単なJavaScriptアプリケーションならgh-pagesへデプロイしてしまって使うの、割とありかなーという感じが最近しています。

## デバッガの話
AngularJSを結構書くうちにデバッガがほしくなってきたんですが、(AngularJSのテンプレートのデバッグがむずい)メーリングリストとかみてみたら、
[Batarang](https://github.com/btford/angularjs-batarang)というChrome拡張があったので僕はこれを使っています。
これ単体の記事もそのうち書くと思います。

## まとめ
思いの他うまくいった。けどやっぱりプレーンなObject返すSDKを自分で作る方がよいような気がする・・・。
Parse自体は使った感じそんなに悪くなくて、データビュアーとかもちゃんとあるので、個人で運用するようなサービスなら全然いけそうだなーという感じでした。
割とサーバーサイド用意するのがだるくて作る気が起きなかったものとか結構あるので、これを気にいろいろ作ってみるかもしれません。
とにかくクライアントサイド書いてるのがすきーな人にはかなり便利なサービスでした、おすすめです。

## 教訓
ずっとスティーブ・Objective-C・ジョブズしててJavaScript界隈についていけていないので遅れ取り戻さないとやばいですねー。
