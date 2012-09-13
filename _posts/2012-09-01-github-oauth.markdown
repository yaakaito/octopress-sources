---
layout: post
title: "iOSアプリでGithubにOAuthする"
date: 2012-09-01 19:12
comments: true
categories: [iOS, Objective-C, Github]
---

こんにちは！うきょーです。
Githubと連動したアプリを作りたくなったので、GithubのOAuthをiOSアプリでやってみました。

* [OAuth | GitHub API](http://developer.github.com/v3/oauth/)
* [GithubOAuthExample](https://github.com/yaakaito/GithubOAuthExample)

## アプリケーションを登録

[AccountSettings]-> [Applications] -> [Register new application]からアプリケーションを登録します。
この時にCallback URLにカスタムスキームを入れる事ができるので、

```
yourapp://oauth
```

みたいなコールバックを指定します。そうするとIDがもらえるのでこれで登録は終わりです。

## Githubへログインしてもらう

OAuthなので、Safariを開いてacceptしてもらいましょう。パスワード入力してもらってJS使って押すとか、やめましょうね。
発行されたIDのうちClient IDをくっつけて

```
NSString *scope = @"public_repo,gist";
NSURL *url = [NSURL URLWithString:[NSString stringWithFormat:@"https://github.com/login/oauth/authorize?client_id=%@&scope=%@",kClientId, scope]];
[[UIApplication sharedApplication] openURL:url];
```

みたいな具合で、Safariを起動します。
別にUIWebViewでもいいと思いますが、専用のビュー作るのもめんどくさいし、ちゃんとGithubなことを証明するのもだるいので、Safariでいいと思います。

## コールバックを拾ってアクセストークンをリクエスト

カスタムスキームからの起動で`code`がやってくるので、これを持って

```
/login/oauth/access_token
```

へPOSTします。


あとはkey-value形式でトークンが返ってくるので、それを使うだけです。簡単ですね。トークンはちゃんとキーチェインとかに入れてあげましょう。

