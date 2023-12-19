---
title: "【Jetpack Compose】状態の読み取りを延期してパフォーマンスの向上させる"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android", "Jetpack Compose"]
published: false
---

皆さんは Jetpack Compose を使っていますか？Compose はすっかり浸透してきていて、Android 開発者の間ではかなりの人が使っているのではないでしょうか。私も業務で使っていて日々恩恵を受けています。

Compose はとても強力ですが、パフォーマンスについては頭を悩ませることもあるかと思います。 パフォーマンス向上の手段はいくつかありますが、今回は状態の読み取りを延期する方法についてまとめてみたいと思います。

## 最初にまとめ

- Jetpack Compose では状態が変更されると再コンポーズが走る
- 状態を読み取っている箇所に最も近い親コンポーザブルが再コンポーズされる
- 状態の読み取りを延期させることで、再コンポーズの範囲を狭めることができる
- Layout、Drawing フェーズまで状態の読み取りを延期させると Composition フェーズをスキップできる

これらについて詳しく見ていきましょう。

## 再コンポーズのタイミング

まずは再コンポーズのタイミングについて見ていきます。

再コンポーズについては Android Developers で以下のように説明があります。

> 再コンポーズとは、入力が変更されたときにコンポーズ可能な関数を再度呼び出すプロセスです。これは、関数の入力が変更された場合に発生します。新しい入力に基づいて再コンポーズされると、Compose は変更された可能性のある関数またはラムダのみを呼び出し、残りはスキップします。パラメータを変更しない関数またはラムダをすべてスキップすることで、Compose で効率的に再コンポーズできます。

https://developer.android.com/jetpack/compose/mental-model?hl=ja#recomposition


再コンポーズは [State#value](https://developer.android.com/reference/kotlin/androidx/compose/runtime/State#value()) の値が読み取られており、かつその値が変更された場合に発生します。

以下はボタンによって値をカウントアップし、それをテキストとして表示する例です。

```kotlin
@Composable
fun MyAppScreen() {
    Log.d("compose-log", "MyAppScreen")
    var count by remember { mutableStateOf(0) }
    Column(
        modifier = Modifier
            .fillMaxSize()
            .wrapContentSize()
    ) {
        Count(count = count)
        Button(onClick = { count += 1 }) {
            Log.d("compose-log", "Button")
            Text(text = "Count Up")
        }
    }
}

@Composable
fun CountText(count: Int) {
    Log.d("compose-log", "CountText")
    Text(
        text = "Count $count"
    )
}
```


`CountText` の引数として `count` の値をそのまま渡しているため、`count` の値が変更されると表示を更新するために再コンポーズが走ります。

![](/images/jetpack-compose-lifecycle/compose-log1.gif)

入力が変化していない場合は再コンポーズがスキップされます。Button のログが初回しか出力されていないのは、`Button` コンポーザブルに渡している引数が変化しておらず、再コンポーズがスキップされているためですね。

しかし、画面がカクつくなどパフォーマンスが問題となっている場合は、この再コンポーズが多発している可能性があります。

## 再コンポーズが行われる範囲

さて、再コンポーズによって表示が更新されることが分かりましたが、次は再コンポーズが行われる範囲について触れたいと思います。

結論として、**状態を読み取っている箇所に最も近い親コンポーズが再コンポーズされます**。

先程のComposableは以下のようなUIツリー構造になっていました。

```
MyAppScreen
├── CountText ← ここで count を読み取っている
│   └── Text
└── Button
    └── Text
```

状態である `count` は `CountText` の引数として渡されているため、そこから最も近い親コンポーザブルである `MyAppScreen` が再コンポーズされます。ログに出力されていたのはこのような作りのためですね。

注意点として、`Column` は inline 関数として定義されているため再コンポーズ範囲の判定には使われません。

## 再コンポーズの範囲を狭める

再コンポーズが何を起点として行われるかを理解したところで、状態の読み取りを延期させる方法を見ていきましょう。

以下は `CountText` に渡している引数をIntではなく、ラムダに変更した例です。

```kotlin
@Composable
fun MyAppScreen() {
    Log.d("compose-log", "MyAppScreen")
    var count by remember { mutableStateOf(0) }
    Column(
        modifier = Modifier
            .fillMaxSize()
            .wrapContentSize()
    ) {
        CountText(countProvider = { count })
        Button(onClick = { count += 1 }) {
            Log.d("compose-log", "Button")
            Text(text = "Count Up")
        }
    }
}

@Composable
fun CountText(countProvider: () -> Int) {
    Log.d("compose-log", "CountText")
    Text(
        text = "Count ${countProvider()}"
    )
}
```

引数をラムダにすることで、`count` が読み取られるタイミングを `CountText` 内の `Text` まで延期することができます。こうすることで、`count` が変更されたときに状態を読み取っている箇所に最も近い親コンポーザブルである `CountText` が再コンポーズされるだけで済みます。

```
MyAppScreen
├── CountText 
│   └── Text ← ここでラムダを呼び出して count を読み取っている
└── Button
    └── Text
```

![](/images/jetpack-compose-lifecycle/compose-log2.gif)

引数の渡し方として直感的では無いかもしれませんが、このようにすることで再コンポーズの範囲を狭めることができ、パフォーマンスの向上に繋がります。

## ラムダModifierを使ってフェーズをスキップする

次はラムダModifierについて見ていきましょう。

その前に、Jetpack Compose の3つのフェーズについて軽く触れておきます。

Compose には以下のようなフェーズがあり、それぞれ役割が異なっています。

1. Composition → 何を表示するかを決める
2. Layout → どこに配置するかを決める
3. Drawing → どのように描画するか決める

![](/images/jetpack-compose-lifecycle/compose-three-phases.png)

https://developer.android.com/jetpack/compose/phases?hl=ja

「どこに配置するか」や「どのように描画するか」のみを変更する場合は、ラムダModifierを使って状態の読み取りを延期させることで、フェーズをスキップすることができます。

以下は `Slider` を使って `CountText` の表示位置を移動できるComposableの例です。

```kotlin
@Composable
fun MyAppScreen() {
    Log.d("compose-log", "MyAppScreen")

    var count by remember { mutableStateOf(0) }
    var sliderValue by remember { mutableStateOf(0f) }

    BoxWithConstraints {
        val maxWidth = constraints.maxWidth
        Column(
            modifier = Modifier
                .fillMaxSize()
                .wrapContentSize()
        ) {
            CountText(
                countProvider = { count },
                xOffsetProvider = { (sliderValue * maxWidth).toInt() }
            )
            Button(onClick = { count += 1 }) {
                Log.d("compose-log", "Button")
                Text(text = "Count Up")
            }
            Slider(value = sliderValue, onValueChange = { sliderValue = it })
        }
    }
}

@Composable
fun CountText(countProvider: () -> Int, xOffsetProvider: () -> Int) {
    Log.d("compose-log", "CountText")
    val xOffset = with(LocalDensity.current) { xOffsetProvider().toDp() }
    Text(
        modifier = Modifier.offset(x = xOffset, y = 0.dp),
        text = "Count ${countProvider()}"
    )
}
```

![](/images/jetpack-compose-lifecycle/compose-log3.gif)

これだとスライダーを動かすたびに `CountText` が再コンポーズされてしまいます。今回は位置を変更しているただけなのでラムダModiferの利用を検討できそうです。

位置の変更に `Modifier.offset` を使っていますが、ラムダを渡すバージョンがあるためそちらを使います。

```kotlin
@Composable
fun MyAppScreen() {
    Log.d("compose-log", "MyAppScreen")

    var count by remember { mutableStateOf(0) }
    var sliderValue by remember { mutableStateOf(0f) }

    BoxWithConstraints {
        val maxWidth = constraints.maxWidth
        Column(
            modifier = Modifier
                .fillMaxSize()
                .wrapContentSize()
        ) {
            CountText(
                countProvider = { count },
                xOffsetProvider = { (sliderValue * maxWidth).toInt() }
            )
            Button(onClick = { count += 1 }) {
                Log.d("compose-log", "Button")
                Text(text = "Count Up")
            }
            Slider(value = sliderValue, onValueChange = { sliderValue = it })
        }
    }
}

@Composable
fun CountText(countProvider: () -> Int, xOffsetProvider: () -> Int) {
    Log.d("compose-log", "CountText")
    Text(
        modifier = Modifier.offset { IntOffset(xOffsetProvider(), 0) }, // ラムダを渡す形式の offset を使う
        text = "Count ${countProvider()}"
    )
}
```


![](/images/jetpack-compose-lifecycle/compose-log4.gif)

スライダーを動かしてもログが出力されなくなりました。`Modifier.offset` に渡したラムダは Layout フェーズで呼び出されます。つまり、状態の読み取りが Layout フェーズまで延期されるため、Composition フェーズがスキップされます。

ラムダを渡すことで Composition またh Layout フェーズをスキップできる Modifier は他にも以下のようなものがあります。

- [absoluteOffset](https://developer.android.com/reference/kotlin/androidx/compose/ui/Modifier#(androidx.compose.ui.Modifier).absoluteOffset(kotlin.Function1))
- [graphicsLayer](https://developer.android.com/reference/kotlin/androidx/compose/ui/Modifier#(androidx.compose.ui.Modifier).graphicsLayer(kotlin.Function1))
- [drawBehind](https://developer.android.com/reference/kotlin/androidx/compose/ui/Modifier#(androidx.compose.ui.Modifier).drawBehind(kotlin.Function1))

## 最後に

以上、再コンポーズのタイミングと範囲から、状態の読み取りを延期させる方法について見てきました。
今回紹介した方法は、実際に効果があるかどうかは状況によって異なると思います。しかし、パフォーマンスに問題がある場合は試してみる価値はあると思いますので、頭の片隅に置いておくといざというときに取れる選択肢が増えそうです。

皆様のパフォーマンス向上に役立てば幸いです。

## 参考記事

- [Jetpack Compose のフェーズ  \|  Android Developers](https://developer.android.com/jetpack/compose/phases?hl=ja)
- [Performance With Jetpack Compose — Part 1 \| by Udit Verma \| ProAndroidDev](https://proandroiddev.com/performance-with-jetpack-compose-part-1-4867882949e7)
