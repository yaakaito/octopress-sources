---
layout: post
title: "ビューテストで便利なAlertNinjaを作りました"
date: 2012-09-05 19:45
comments: true
categories: [iOS, Objective-C, Testing, Library, AlertNinja] 
---

こんにちは！うきょーです！前回ふざけてましたが、今回は真面目にライブラリを作りました。
といっても、毎回書くのはめんどくさいのでまとめとくか程度です。

みなさん！iOSのテスト書いてますか？？？
ViewControllerなんかのテストを書いてるときに、例えばGHUnitなんかを使ってたりすると、
UIAlertViewがカジュアルに飛び出てきてウザイですよね＾ー＾ー＾ー＾

僕はUIAlertViewは、適当なラッパークラスを作って、そこを差し替えて出ないようにすることが多いんですが、
テストの為のクラスを作ってる様なものになってしまう上に、毎回書くのがだるポヨ・・・。

なのでついでだしテストも出来るようにしよう、ってことで、UIAlertViewを非表示にしつつ、スパイするライブラリを作りました。

* [AlertNinja](https://github.com/yaakaito/AlertNinja)

## AlertNinjaの機能
さっきも書きましたが、UIAlertViewを非表示にする、とスパイするの２つの機能を持っています。
この二つの機能はほとんど同時に使う事が多くなると思います。

使い方はこんな感じ、適当なViewControllerがこんな感じでshowDialogというメソッドを持っていたとすると、
```objective-c
- (void)showDialog {  
  UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Ninja"
                                                  message:@"doron"
                                                 delegate:nil
                                        cancelButtonTitle:@"YES"
                                        otherButtonTitles:nil];
  [alert show];
}
```

テストを書くときにどこかからこれを読んでいると、アラートが表示されてしまいますね。
なのでAlertNinjaを使って、アラートがでる可能性のあるところを囲みます。

```objective-c
- (void)testDialog
{   
    [[UIAlertView ninja] spy];
    [viewController showDialog];
    [[UIAlertView ninja] complete];

}
```

`spy`でスタートして、`complete`で終了です、なんか忍者っぽい感じにしたかったんです！！！
こうすると、まずアラートの表示をなかった事にできます。

次はどんなアラートか出たか知りたいですよね、出てないかもしれません。
これは`report`というものを取得することで検証できます。

```objective-c
UIAlertView *alert = [[[[UIAlertView ninja] report] showedAlerts] lastObject];
STAssertEqualObjects(@"Ninja", alert.title, @"alert title is Ninja");
```

`showedAlerts`は`spy`されてから表示されたUIAlertViewのリストです。(今のところこの機能しかないです。)
なので、これの`count`が0だったらアラートはなかったことになりますし、そうでなければ、その中身を検証できます。
今回の例では`title`が`Ninja`なアラートが表示されるはず、というテストになっていますね。

つなげるとこんな感じです。

```objective-c
- (void)testDialog
{   
    [[UIAlertView ninja] spy];
    [viewController showDialog];
    UIAlertView *alert = [[[[UIAlertView ninja] report] showedAlerts] lastObject];
    STAssertEqualObjects(@"Ninja", alert.title, @"alert title is Ninja");
    [[UIAlertView ninja] complete];

}
```

### confirmもできるよ！
ボタンを何個か設定して、ここを押したい、みたいなテストにも対応できます。

```objective-c
- (void)showConfirm {    
    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Ninja"
                                                    message:@"Are you Ninja ?"
                                                   delegate:self
                                          cancelButtonTitle:@"NO"
                                          otherButtonTitles:@"YES", @"I'm Kunoichi", nil];
    [alert show];
}

- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    self.calledClickedButtonAtIndex = YES;
    if(buttonIndex == 0) {
        self.result = @"NO";
    }
    else if(buttonIndex == 1) {
        self.result = @"YES";
    }
    else if(buttonIndex == 2) {
        self.result = @"Kunoichi";
    }
}
```

こうなってるやつに・・・

```objective-c
- (void)testConfirm
{
    [[[UIAlertView ninja] spy] andSelectIndexAt:2];
    [viewController showConfirm];
    UIAlertView *alert = [[[[UIAlertView ninja] report] showedAlerts] lastObject];

    STAssertEqualObjects(@"Ninja", alert.title, @"alert title is Ninja");
    STAssertEqualObjects(@"Kunoichi", viewController.result, @"result is 'Kunoichi'");

    [[UIAlertView ninja] complete];
}
```

こんな感じで、`spy`に続けて`andSelectIndexAt`でどのインデックスのボタンを押すかを指定することができます。何も設定しないとキャンセルボタン扱いになります。
もちろんDelegateも呼ばれていて、さっきのViewControllerにはさらにこんなのが続いていて、

```objective-c
- (void)willPresentAlertView:(UIAlertView *)alertView {
    self.calledWillPresent = YES;
}

- (void)didPresentAlertView:(UIAlertView *)alertView {
    self.calledDidPresent = YES;
}

- (void)alertView:(UIAlertView *)alertView willDismissWithButtonIndex:(NSInteger)buttonIndex {
    self.calledWillDismiss = YES;
}

- (void)alertView:(UIAlertView *)alertView didDismissWithButtonIndex:(NSInteger)buttonIndex {
    self.calledDidDismiss = YES;
}
```

全体でこんなテストが通るようになっています。
```objective-c
- (void)testConfirm
{
    [[[UIAlertView ninja] spy] andSelectIndexAt:2];
    [viewController showConfirm];
    UIAlertView *alert = [[[[UIAlertView ninja] report] showedAlerts] lastObject];

    STAssertEqualObjects(@"Ninja", alert.title, @"alert title is Ninja");
    STAssertTrue(viewController.calledWillPresent, @"called will present delegate method");
    STAssertTrue(viewController.calledDidPresent, @"called did present delegate method");
    STAssertTrue(viewController.calledWillDismiss, @"called will dismiss delegate method");
    STAssertTrue(viewController.calledDidDismiss, @"called did dismiss delegate method");
    STAssertTrue(viewController.calledClickedButtonAtIndex, @"called did clicked button at Index");
    STAssertEqualObjects(@"Kunoichi", viewController.result, @"result is 'Kunoichi'");

    [[UIAlertView ninja] complete];
}
```

## 流れがテストできる

UIAlertViewを含んだテストができるようになったので、全体としてフィーチャーのテストがし易くなりました。
例えば [NLTHTTPStubServer](https://github.com/yaakaito/NLTHTTPStubServer) と組み合わせると、
「APIにアクセスしたけど、404だったから"そんなものはない"というアラートだす」みたいなテストを結構スマートに書く事ができますね！！！(宣伝)
テストの為の何かをほとんどプロダクトコードに埋め込まなくとも良いのも特徴です。

## TODO

今はこれだけで、以下には対応してない＆やろうと思っているので乞うご期待！

* UIActionSheetも使えるようになる
* UIAlertViewStyleのサポート

他にもこれ必要じゃね、というのがあればIssueなどに投げてください！！！

## というわけで

よろしくお願いします！

* [AlertNinja](https://github.com/yaakaito/AlertNinja)

