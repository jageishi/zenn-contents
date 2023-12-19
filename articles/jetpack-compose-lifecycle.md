---
title: "【Jetpack Compose】状態の読み取りを遅延してパフォーマンスを向上させる"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android", "Jetpack Compose"]
published: false
---

皆さんは Jetpack Compose を使っていますか？ Compose は Android アプリ開発者の間で広く浸透してきて、多くの方が利用しているのではないかと思います。私自身も業務で使っており、その恩恵を日々実感しています。

Compose は非常に強力なツールですが、パフォーマンスの向上に関してはテクニックが必要だったりします。方法はいくつか存在しますが、今回は状態の読み取りを遅延させる手法について紹介していきたいと思います。

## 再コンポーズについて

まずは再コンポーズのタイミングについて軽く触れたいと思います。

再コンポーズについては Android Developers で以下のように説明があります。

> 再コンポーズとは、入力が変更されたときにコンポーズ可能な関数を再度呼び出すプロセスです。これは、関数の入力が変更された場合に発生します。新しい入力に基づいて再コンポーズされると、Compose は変更された可能性のある関数またはラムダのみを呼び出し、残りはスキップします。パラメータを変更しない関数またはラムダをすべてスキップすることで、Compose で効率的に再コンポーズできます。

https://developer.android.com/jetpack/compose/mental-model?hl=ja#recomposition

Compose は [State](https://developer.android.com/reference/kotlin/androidx/compose/runtime/State) の変更を観測します。再コンポーズは、 [State#value](https://developer.android.com/reference/kotlin/androidx/compose/runtime/State#value()) が参照されており、かつその値が変更された場合に発生します。

以下はボタンをタップしてカウントアップするコンポーザブルです。

```kotlin
@Composable
fun MyAppScreen() {
    SideEffect { Log.d("compose-log", "MyAppScreen") }
    var count by remember { mutableStateOf(0) }
    Column(
        modifier = Modifier
            .fillMaxSize()
            .wrapContentSize()
    ) {
        CountText(count = count)
        Button(onClick = { count++ }) {
            SideEffect { Log.d("compose-log", "Button") }
            Text(text = "Count Up")
        }
    }
}

@Composable
fun CountText(count: Int) {
    SideEffect { Log.d("compose-log", "CountText") }
    Text(
        text = "Count $count"
    )
}
```

`CountText` の引数として `count` を渡しているため、`count` が変更されると表示を更新するために `MyAppScreen` が再コンポーズされます。

![](/images/jetpack-compose-lifecycle/compose-log1.gif)

### 再コンポーズのスコープについて

先程のコンポーザブルは以下のようなUIツリー構造になっています。

```
MyAppScreen
└── Column
    ├── CountText ← ここで count を読み取っている
    │   └── Text
    └── Button
        └── Text
```

Compose は、状態の読み取り位置から最も近い親のスコープを見つけて再コンポーズを行います。

上記の例だと、状態である `count` は `CountText` の引数として渡されているため、そこから最も近い親のスコープである `MyAppScreen` が再コンポーズされています。

:::details 注意： Column は inline 関数として定義されており restartable でないコンポーザブルのため、スコープとしては機能しません。再コンポーズされる場合は親コンポーザブルも再コンポーズされます。
https://developer.android.com/jetpack/compose/performance/stability?hl=ja#functions
:::


## 状態の読み取りを子コンポーザブルに移動する

次に状態の読み取りを子コンポーザブルに移動して、再コンポーズされる範囲を抑えてみます。

以下は、 `CountText` に渡している引数を Int ではなくラムダに変更した例です。

```kotlin
@Composable
fun MyAppScreen() {
    SideEffect { Log.d("compose-log", "MyAppScreen") }
    var count by remember { mutableStateOf(0) }
    Column(
        modifier = Modifier
            .fillMaxSize()
            .wrapContentSize()
    ) {
        CountText(countProvider = { count })
        Button(onClick = { count++ }) {
            SideEffect { Log.d("compose-log", "Button") }
            Text(text = "Count Up")
        }
    }
}

@Composable
fun CountText(countProvider: () -> Int) {
    SideEffect { Log.d("compose-log", "CountText") }
    Text(
        text = "Count ${countProvider()}"
    )
}
```

引数をラムダにすることで、`count` が読み取られるタイミングを `CountText` 内の `Text` まで遅延することができます。

```
MyAppScreen
└── Column
    ├── CountText 
    │   └── Text ← ここでラムダを呼び出して count を読み取っている
    └── Button
        └── Text
```

こうすることで、`count` が変更されたときに状態を読み取っている箇所から最も近いスコープである `CountText` が再コンポーズされるだけで済みます。

![](/images/jetpack-compose-lifecycle/compose-log2.gif)

## ラムダ版のModifierを使ってフェーズをスキップする

次はラムダ版のModifierについて見ていきます。その前に Jetpack Compose の3つのフェーズについて軽く触れておきます。

Compose では、以下のように役割が異なる3つのフェーズを経てUIが表示されます。

1. Composition → 何を表示するかを決める
2. Layout → どこに配置するかを決める
3. Drawing → どのように描画するかを決める

![](/images/jetpack-compose-lifecycle/compose-three-phases.png)

https://developer.android.com/jetpack/compose/phases?hl=ja

「どこに配置するか」や「どのように描画するか」のみを変更する場合は、ラムダ版Modifierを使って状態の読み取りを遅延させることで、フェーズをスキップすることができます。

以下は `Slider` を使って `CountText` の表示位置を移動できるコンポーザブルです。

```kotlin
@Composable
fun MyAppScreen() {
    SideEffect { Log.d("compose-log", "MyAppScreen") }

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
            Button(onClick = { count++ }) {
                SideEffect { Log.d("compose-log", "Button") }
                Text(text = "Count Up")
            }
            Slider(value = sliderValue, onValueChange = { sliderValue = it })
        }
    }
}

@Composable
fun CountText(countProvider: () -> Int, xOffsetProvider: () -> Int) {
    SideEffect { Log.d("compose-log", "CountText") }
    val xOffset = with(LocalDensity.current) { xOffsetProvider().toDp() }
    Text(
        modifier = Modifier.offset(x = xOffset, y = 0.dp),
        text = "Count ${countProvider()}"
    )
}
```

![](/images/jetpack-compose-lifecycle/compose-log3.gif)

これだとスライダーを動かすたびに `CountText` が再コンポーズされてしまいます。

今回、位置の変更に使っている`Modifier.offset`にはラムダ版と非ラムダ版があります。

```kotlin
// ラムダ版
fun Modifier.offset(offset: Density.() -> IntOffset): Modifier

// 非ラムダ版
fun Modifier.offset(x: Dp = 0.dp, y: Dp = 0.dp): Modifier
```

ラムダ版の`Modifier.offset`を使って`CountText`を書き換えると以下のようになります。

```kotlin
@Composable
fun CountText(countProvider: () -> Int, xOffsetProvider: () -> Int) {
    SideEffect {Log.d("compose-log", "CountText")}
    Text(
        modifier = Modifier.offset { IntOffset(xOffsetProvider(), 0) },
        text = "Count ${countProvider()}"
    )
}
```

![](/images/jetpack-compose-lifecycle/compose-log4.gif)

スライダーを動かしてもログが出力されなくなりました。`Modifier.offset` に渡したラムダは Layout フェーズで参照されます。つまり、状態の読み取りが Layout フェーズまで遅延されるため、Composition フェーズがスキップされます。

ラムダを渡すことでフェーズをスキップできる Modifier は他にも以下のようなものがあります。

- [absoluteOffset](https://developer.android.com/reference/kotlin/androidx/compose/ui/Modifier#(androidx.compose.ui.Modifier).absoluteOffset(kotlin.Function1))
- [graphicsLayer](https://developer.android.com/reference/kotlin/androidx/compose/ui/Modifier#(androidx.compose.ui.Modifier).graphicsLayer(kotlin.Function1))
- [drawBehind](https://developer.android.com/reference/kotlin/androidx/compose/ui/Modifier#(androidx.compose.ui.Modifier).drawBehind(kotlin.Function1))

## おわりに

以上、状態の読み取りを遅延させる方法についてでした。
今回紹介した方法が実際に効果を発揮するかどうかは、状況によって異なります。また、パフォーマンスの早すぎる最適化は徒労に終わってしまうこともあるため、まずは基本的な対応に留めておき、問題が生じた際にはボトルネックとなっている箇所を特定して対応していくのが効率的だと思います。
今回紹介したような方法を頭の片隅に置いておくと、そのような時にとれる選択肢が増えるかもしれません。

## 参考記事

- [Jetpack Compose のフェーズ  \|  Android Developers](https://developer.android.com/jetpack/compose/phases?hl=ja)
- [Performance With Jetpack Compose — Part 1 \| by Udit Verma \| ProAndroidDev](https://proandroiddev.com/performance-with-jetpack-compose-part-1-4867882949e7)
- [おすすめの方法を実践する  \|  Jetpack Compose  \|  Android Developers](https://developer.android.com/jetpack/compose/performance/bestpractices?hl=ja)
