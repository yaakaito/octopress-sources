---
layout: post
title: "dequeueReusable~の実装とテスト"
date: 2012-08-19 02:35
comments: true
categories: [Objective-C, iOS, Testing, QuickCheck, NLTQuickCheck]
---

こんにちは！うきょーです。UITableViewに実装されている`dequeueReusableCellWithIdentifier`と同等の機能を持ったものを開発したいんだけど的な話です。
dataSourceに似たのインターフェイスを持ってビューを実装するときに、不特定個数のものに対してインスタンスをすべて生成するわけにも行かないので、ある程度制御してあげる必要があります。
今回はUIScrollView上に構築していく前提になっています。シンタックスハイライターを作ろうとしているので、使い回して描画される対象になるのは、行数表示とコード一行分の表示です。

コードはいつも通りgithubにあります。

* [BGSyntaxHighlighter](https://github.com/yaakaito/BGSyntaxHighlighter)
  * 注)ライブラリとしては未完成です

## 実装方法を考える
さらっと思いつくところではリングバッファのようにしておけばよい気がしますね。
ATPagingViewが似たような機能を実装していたのでコード読んでみたのですが、こちらはページング終了時にリサイクル用のSetに取っておいて、
次の表示タイミングでそこから適当な物をもっていく、という方式でした。微妙にリングバッファと違う・・・。
リングバッファっぽい感じでいけそう？と考えていたのですが、リングバッファにしてしまうと、画面の大きさに合わせてバッファの数を調整しなおさければいけなかったり、
今回は中で簡潔している構造になっているのでよいのですが、APIを外に向けたときに、identifier関連でめんどくなりそうだなーという気がしたので、ATPagingViewがやっているような方法を採用しました。
やることとしては、

* 初回時に必要な分のビューを作る
* スクロールにあわせて必要なビューの差分を取って作る
* 必要なビューを使い回す対象にする

案外、簡単そうに見えますね。

## UIScrollViewベースで実装するときの注意点
UIScrollViewで実装する場合には、UITableViewほど正確にスクロールの位置がとれる訳ではないのがポイントというか、気をつけるところです。
今回はめんどくさいのでやらなかったのですが、ある程度余裕を持って描画してあげないと、下の方のビューが欠けてしまったりします。
適当な余裕をもって描画してあげるといい感じになると思います。

## 実装していこう！
というわけで実装していきましょう。

#### 下準備
まずはdequeueReusableのインターフェイスを作ります。今回は行数表示とコード一行分の表示を分けているので、2つ分必要です。
が、実装としては両方同じなので、行数表示だけの例を出していきます。全文を確認したい方はgithubからどうぞ！

```objective-c
- (BGSyntaxHighlightLineNumberView*)dequeueReusableLineNumberView {
    
    if(self.recycleLineNumberViews && [self.recycleLineNumberViews count] > 0) {
        BGSyntaxHighlightLineNumberView *view = [self.recycleLineNumberViews anyObject];
        [self.recycleLineNumberViews removeObject:view];
        return view;
    }
    return nil;
}
```

`recycleLineNumberViews`は`NSMutableSet`で、ここに再利用可能なビューを入れて行きます。
このメソッドは利用可能なビューが存在すれば、そこから適当なビューを抜き出してきて返す、という具合です。
実際にビューに対しての描画処理をここでやってしまうと残念な感じになってしまうので、依存しないように気をつけましょう。
次のここからビューの取得を試みて、なければ新しいものを作って、描画を行うメソッドです。
UITableViewで言えば、`cellForRowAtIndexPath:`にあたる部分で、iOSエンジニアなら見覚えのある感じだと思います。

```objective-c
- (BGSyntaxHighlightLineNumberView*)lineNumberViewAtRow:(NSInteger)row {
    
    BGSyntaxHighlightLineNumberView *view = [self dequeueReusableLineNumberView];
    if(!view) {
        view = [[BGSyntaxHighlightLineNumberView alloc] initWithFrame:CGRectZero];
    }
    view.lineNumber = row;
    return view;
}
```

これで大体の準備ができました。`BGSyntaxHighlightLineNumberView`が必要なときは、このメソッド経由で取得すれば、
再利用できるときは再利用を、ビューが足りずに新しく作る必要があるときは新しくビューを作ってくれます。

### UIScrollViewへ表示する
では、これを使って実際に`UIScrollView`へ行数を表示してみましょう。
初回のタイミングというか、最初に必要なものが決まるタイミングで必要な分だけビューを作ります。
(今回は`layoutSubViews`なんですが、レイアウト以外をここでやるの微妙な感じがするので、こっちの方がよくね？というのがあれば教えてください。)

```objective-c
- (void)layoutSubviews {
    // ...     
    
    for(NSInteger i = 0; i < [self.codeObject numberOfCodeLines]; i++) {
        [self makeAndLayoutLineAtRow:i + 1];
        
        if((i + 1) * kLineHeight >= self.frame.size.height) {
            self.viewingLinesRange = NSMakeRange(0U, i);
            break;
        }
    }

    // ...
    
}
```

必要な分だけビューを作って、初期表示に必要ない部分になったらどこまで描画したかを記録しbreakします。
`makeAndLayoutLineAtRow:`は指定した行のビューを生成してくれるメソッドで、中身はこんな感じです。

```objective-c
(void)makeAndLayoutLineAtRow:(NSInteger)row {
    
    BGSyntaxHighlightLineNumberView *lineNumberView = [self lineNumberViewAtRow:row];
    [self.lineNumberViews addObject:lineNumberView];
    [self.lineNumberScrollView addSubview:lineNumberView];
    lineNumberView.frame = CGRectMake(0, kLineHeight * (row-1), kLineNumberWidth, kLineHeight);
}
```

ここまでで、とりあえず最初の表示が作れました、スクロールにあわせて必要な分を生成するようにしましょう。
`UIScrollViewDelegate`の`scrollViewDidScroll:`で、

```objective-c
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {

    // ...
    
    CGFloat y = scrollView.contentOffset.y;
    if(y <= 0 || y + scrollView.frame.size.height > [self.codeObject numberOfCodeLines] * kLineHeight) {
        return;
    }
    
    // いらないビューを再利用対象にする
    [self recycleLinesOfOutsideFromRangeY:NSMakeRange(scrollView.contentOffset.y, scrollView.frame.size.height)];

    /*
      差分を計算して必要なビューをmakeAndLayoutLineAtRow:で描画
    */  
    
```

差分の計算ロジックなんかは長くなってしまうので、省略しています。表示範囲内にはいっているけれど、まだ表示されてないビューがある場合に表示するロジックです。
`recycleLinesOfOutsideFromRangeY:`は、名前がよくわからなかったので微妙なのですが、スクロール後に必要なくなったものを`removeFromSuperView`して、再利用対象にするメソッドです。
一見作ってから削除、でもそんなに変わらない気がしますが、先に削除しておくと、削除されたものをそのまま使い回すことができるので、少しメモリが節約できます。
`recycleLinesOfOutsideFromRangeY:`を省略して簡単に乗せておくと、

```objective-c
- (void)recycleLinesOfOutsideFromRangeY:(NSRange)range {
    
    NSMutableArray *addingRecycleLineNumberViews = [NSMutableArray array];
    for (NSUInteger i = 0, length = [self.lineNumberViews count]; i < length; i++) {
       /*
          範囲外のものを抽出
        */ 
    }
    [self.recycleLineNumberViews addObjectsFromArray:addingRecycleLineNumberViews];
    [self.lineNumberViews removeObjectsInArray:addingRecycleLineNumberViews];
}
```
という具合です。

### 実際どれくらいのビューを必要とするのか
ここまでで大体実装ができた訳ですが、実際にどれくらいのビューが必要になるのでしょうか。
今回の例では一行の高さが20pxなので、縦を460px確保したとすると、初期状態で460/20で23個必要になります。
スクロールにあわせて描画を行って行くと、初回で何個かallocされれ、あとは大抵24~27個程度のビューのみで構成することがきでています。


## テストを書く
さて、次はテストを書きましょう。`dequeueReusable~`のテストとして必要な項目は、

* あるスクロール位置でのビューが想定した通りにレンダリングされているか
* 適当なところにスクロールしたときにビューを使い回しているか

の2つでしょうか。まずは1つ目の方からテストしてみましょう。
説明を分かり易くするために、ここからビューの高さを100px(ぴったりの場合ビューは5個)で統一していきます。

### 想定したレンダリングになっているかテスト
キチンとレイアウトが出来ているかは別のテスト(`dequeueReusable~`を無視したテスト)に任せるとして、スクロールした位置によってビューの枚数がキチンとあっているかを確認しましょう。
まずは、ぴったりと高さの倍数で移動したケース。

```objective-c
- (void)testRenderingViews {

    view.frame = CGRectMake(0, 0, 320, 100);
    [view layoutSubviews];
    GHAssertEquals(5U, [view.lineNumberViews count], @"100/20=5で初期段階で5個ある");
    GHAssertEquals(5U, [view.lineViews count], @"100/20=5で初期段階で5個ある");
    
    [view.lineNumberScrollView scrollRectToVisible:CGRectMake(0, 180, 320, 100) animated:NO];
    GHAssertEquals(5U, [view.lineNumberViews count], @"ぴったりで移動するのでやっぱり5個ある");
    GHAssertEquals(5U, [view.lineViews count], @"ぴったりで移動するのでやっぱり5個ある");
}
```

こっちは簡単ですね。次に10pxくらいずれた場合、この場合は一番上と下に半分づつのビューが必要なので、全体で6個になります、

```objective-c
- (void)testMisalignRenderingVies {
    
    view.frame = CGRectMake(0, 0, 320, 100);
    [view layoutSubviews];
    
    [view.lineNumberScrollView scrollRectToVisible:CGRectMake(0, 190, 320, 100) animated:NO];
    GHAssertEquals(6U, [view.lineNumberViews count], @"すこしずれて移動するので下に1個余分に追加されて6個になる");
    GHAssertEquals(6U, [view.lineViews count], @"すこしずれて移動するので下に1個余分に追加されて6個になる");
}
```

こんな感じです。

### 適当なところにスクロールした場合でのビューの個数をテスト
どこにスクロールしたときも、ビューを使い回して一定以上の数のビューを生成しないのが今回の目的なので、これをテストします。
自分でたくさんいろんなところにスクロールするテストケースを書いてもいいですが、こういう時はQuickCheckが適用できます。
Objective-CのQuickCheckライブラリは、NLTQuickCheckをご利用ください。(宣伝)

* [NLTQuickCheck](https://github.com/yaakaito/NLTQuickCheck)

標準のdoubleArbitraryでは少し大きいので、doubleArbitaryの`quadraticGenWithA:B:C:`へ渡す数値を調整して、大体+-0~600pxくらいの範囲の値を作ってくれるArbitraryを定義します。

```objective-c
@implementation NSNumber (BGArbitrary)
+ (id)scrollYArbitrary {
    NLTQGen *quadratic = [NLTQGen quadraticGenWithA:1<<16 b:1 c:1];
    NLTQGen *doubleGen = [NLTQGen genWithGenerateBlock:^id(double progress, int random) {
        NLTQGen *chooser = [NLTQGen randomGen];
        [chooser resizeWithMinimumSeed:-random maximumSeed:+random];
        random = [[chooser valueWithProgress:progress] intValue];
        int place = [[NSString stringWithFormat:@"%d",random] length] - 2;
        int l = [[NLTQStandardGen standardGenWithMinimumSeed:2 maximumSeed:place] currentGeneratedValue];
        int base = 10;
        for (int i = 1; i < l; i++) {
            base *= 10;
        }
        return [NSNumber numberWithDouble:(double)random/(double)base];
        
    }];
    [doubleGen bindingGen:quadratic];
    return doubleGen;    
}
@end
```

このArbitraryを使って、さっきのテストと同じ要領で適当な場所へスクロールさせます。

```objective-c
- (BOOL)propRecycleLogic:(NSNumber*)y {
    view = [[BGSyntaxHighlightView alloc] initWithFrame:CGRectZero];
    codeObject = [[BGCodeObject alloc] initWithCodeString:[NSBundle codeStringForResouce:@"mockLongObjective-C" ofType:@"txt"]];
    view.codeObject = codeObject;
    view.frame = CGRectMake(0, 500, 320, 100);
    [view layoutSubviews];
    [view.lineNumberScrollView scrollRectToVisible:CGRectMake(0, [y floatValue], 320, 100) animated:NO];
    GHTestLog(@"%d", [view.lineNumberViews count]);
    return [view.lineNumberViews count] < 8; // 3つくらいは許容できる
}

- (void)testRecycle {
    NLTQTestable *testable = [NLTQTestable testableWithPropertySelector:@selector(propRecycleLogic:) 
                                                                 target:self
                                                            arbitraries:[NSNumber scrollYArbitrary], nil];
    [testable verboseCheck];
    GHAssertTrue([testable success], @"%@", [testable prettyReport]);
}
```

アサーションは`return [view.lineNumberViews count] < 8;`で、ピッタリが5個なので大体8個あれば十分使い回しきれるだろう、という風に書きます。

## まとめ

これで`dequeueReusable~`っぽいものが実装できました。
ビューが多くなってしまうが、不可視なビューが存在する、という状況では実装しておいて損はない機能です。
ただやってみて実装コストは少し高めだなーと感じたので、その辺は相談、という感じでしょうか。
