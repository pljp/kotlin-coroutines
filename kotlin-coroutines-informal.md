# Kotlinコルーチン (Coroutines for Kotlin - Revision 3.2)

* **Type**: Informal description
* **Author**: Andrey Breslav
* **Contributors**: Vladimir Reshetnikov, Stanislav Erokhin, Ilya Ryzhenkov, Denis Zharkov, Roman Elizarov
* **Status**: Implemented in Kotlin 1.1.0

##要約

これはKotlinのコルーチンの説明です。この概念は次のものとして知られているか、部分的に含まれています。

- ジェネレーター/yield
- async/await
- 部分継続/限定継続

ゴール：

- Futureなどのリッチなライブラリの特定の実装に依存しない。
- "async/await"ユースケースと "ジェネレータブロック"を等しくカバーする。
- 異なる既存の非同期APIのラッパーとしてKotlinコルーチンを利用できるようにする
  （Java NIO、Futureのさまざまな実装など）。

## 目次

* [ユースケース](#ユースケース)
  * [非同期計算](#非同期計算)
  * [Futures](#futures)
  * [ジェネレーター](#ジェネレーター)
  * [非同期UI](#非同期UI)
  * [その他のユースケース](#その他のユースケース)
* [コルーチンの概要](#コルーチンの概要)
  * [コルーチンの実験的状態](#コルーチンの実験的状態)
  * [用語](#用語)
  * [継続インタフェース](#継続インタフェース)
  * [サスペンド関数](#サスペンド関数)
  * [コルーチンビルダー](#コルーチンビルダー)
  * [コルーチンコンテキスト](#コルーチンコンテキスト)
  * [継続インターセプター](#継続インターセプター)
  * [制限付きサスペンド](#制限付きサスペンド)
* [その他の例](#その他の例)
  * [コールバックのラッピング](#コールバックのラッピング)
  * [futureの作成](#futureの作成)
  * [ノンブロッキングスリープ](#ノンブロッキングスリープ)
  * [協調シングルスレッドマルチタスキング](#協調シングルスレッドマルチタスキング)
  * [非同期シーケンス](#非同期シーケンス)
  * [チャネル](#チャネル)
  * [ミューテックス](#ミューテックス)
* [詳細トピック](#詳細トピック)
  * [リソース管理とGC](#リソース管理とGC)
  * [並行処理とスレッド](#並行処理とスレッド)
  * [非同期プログラミングスタイル](#非同期プログラミングスタイル)
* [実装の詳細](#実装の詳細)
  * [継続渡しスタイル](#継続渡しスタイル)
  * [状態マシン](#状態マシン)
  * [サスペンド関数のコンパイル](#サスペンド関数のコンパイル)
  * [Coroutine intrinsics](#coroutine-intrinsics)
* [改訂履歴](#revision-history)
  * [Changes in revision 3.2](#changes-in-revision-32)
  * [Changes in revision 3.1](#changes-in-revision-31)
  * [Changes in revision 3](#changes-in-revision-3)
  * [Changes in revision 2](#changes-in-revision-2)

## ユースケース

コルーチンは_中断可能な計算_のインスタンスとして考えることができます。ある時点で中断し、後で別のスレッドで実行を再開することができます。
コルーチンはお互いを呼び出す（そしてデータを前後に渡す）ことで、協力的なマルチタスキングのための機構を形成することができます。

### 非同期計算

コルーチンの動機付けの最初のクラスは、非同期計算です（C#やその他の言語ではasync/awaitで処理されます）。
そのような計算がコールバックでどのように行われるかを見てみましょう。 インスピレーションとして、非同期I/O（以下のAPIは簡略化されています）を取ります:

```kotlin
// 非同期に `buf`を読み込み、完了したらラムダを実行
inChannel.read(buf) {
    // このラムダは読み込みが完了したときに実行される
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)

    // `buf`から非同期的に書き込み、完了したらラムダを実行
    outChannel.write(buf) {
        // このラムダは、書き込みが完了したときに実行される
        ...
        ...
        outFile.close()
    }
}
```

ここではコールバック内にコールバックがあり、多くの定型文から私たちを救っていることに注意してください（コールバックに `buf`パラメータを明示的に渡す必要はありません）。
インデントレベルは毎回増加しています。1つ以上のネストレベルで発生する可能性のある問題を簡単に予測することができます（JavaScriptでこれでどれくらいの人が苦しんでいるかを知るために「コールバック地獄」についてググってください）。

これと同じ計算はコルーチンとして直接的に表現できます（I/O APIをコルーチンの要件に適合させたライブラリがあれば）:

```kotlin
launch(CommonPool) {
    // 非同期で読み込んでいる間サスペンドする
    val bytesRead = inChannel.aRead(buf)
    // 読み込みが完了したときだけこの行に行く
    ...
    ...
    process(buf, bytesRead)
    // 非同期で書き込んでいる間サスペンドする
    outChannel.aWrite(buf)
    // 書き込みが完了したときだけこの行に行く
    ...
    ...
    outFile.close()
}
```

ここでの `aRead()`と `aWrite()`は特別な_サスペンド関数_です。実行を_サスペンド_し、呼び出しが完了すると_リジューム_します。
`aRead()`の後のすべてのコードがラムダでラップされ、コールバックとして `aRead()`に渡され、 `aWrite()`に対して同じことが行われたと想像するだけで、 私たちはこのコードが上記と同じで、より読み易いことが分かります。

これはコルーチンを非常に一般的な方法でサポートするという私たちの明白な目標です。この例では、 `launch {}`、 `.aRead（）`、 `.aWrite（）`は単にコルーチンを扱う**ライブラリ関数です**:
`aRead`/`aWrite`は暗黙的に_継続_(単なる一般的なコールバック)を受け取る特殊な_サスペンド関数_ですが、`launch`(_コルーチンのビルダー_)はコルーチンをコンテキストの中で(この例では `CommonPool`コンテキストが使用されています)でビルドして起動します。

> `launch {}`のライブラリコードは[コルーチンビルダー](#coroutine-builder)セクションに、 `.aRead()`のライブラリコードは[コールバックのラッピング](#wrapping-callbacks)セクションに示されています。

明示的に渡されたコールバックでは、ループの途中で非同期呼び出しを行うことは難しいかもしれませんが、コルーチンでは完全に普通のことです:

```kotlin
launch(CommonPool) {
    while (true) {
        // 読み込んでいる間サスペンドする
        val bytesRead = inFile.aRead(buf)
        // 読み込みが終わったら続ける
        if (bytesRead == -1) break
        ...
        process(buf, bytesRead)
        // 書き込んでいる間サスペンドする
        outFile.aWrite(buf)
        // 書き込みが終わったら続ける
        ...
    }
}
```

コルーチンで例外を処理することはもう少し便利だと想像できるでしょう。

### Futures

非同期計算を表現する別のスタイルがあります: through futures（そして近い関係のpromise）。
画像にオーバーレイを適用するために、ここでは架空のAPIを使用します。

```kotlin
val future = runAfterBoth(
    asyncLoadImage("...original..."), // Futureを作る
    asyncLoadImage("...overlay...")   // Futureを作る
) {
    original, overlay ->
    ...
    applyOverlay(original, overlay)
}
```

コルーチンでは、これは次のように書き直すことができます。

```kotlin
val future = future {
    val original = asyncLoadImage("...original...") // Futureを作る
    val overlay = asyncLoadImage("...overlay...")   // Futureを作る
    ...
    // 画像の読み込みを待つ間中断し、両方が読み込まれたときに
    // `applyOverlay(...)`を実行する
    applyOverlay(original.await(), overlay.await())
}
```

> `future{}`のライブラリコードは[futureの作成](#building-futures)セクションに、 `.await()`のライブラリコードは[サスペンド関数](#suspending-functions)に示されています。

繰り返しますが、字下げが少なく自然な構成ロジック(ここでは示しませんが例外処理も)で、futureをサポートするための特別なキーワード(C#, JSなどの言語での`async`や`await`のような)もありません。
`future{}`や`.await()`はただのライブラリ関数です。

### ジェネレーター

コルーチンの別の典型的な使用例は、遅延計算シーケンス(C#、Python、その他多くの言語で`yield`によって処理される)です。
このようなシーケンスは、一見するとシーケンシャルなコードで生成できますが、実行時には要求された要素のみが計算されます。

```kotlin
// 推定される型はSequence<Int>
val fibonacci = buildSequence {
    yield(1) // 最初のフィボナッチ数
    var cur = 1
    var next = 1
    while (true) {
        yield(next) // 次のフィボナッチ数
        val tmp = cur + next
        cur = next
        next = tmp
    }
}
```

このコードは、潜在的に無限にある[フィボナッチ数](https://en.wikipedia.org/wiki/Fibonacci_number)の遅延`シーケンス`を作成します([Haskelの無限リスト](http://www.techrepublic.com/article/infinite-list-tricks-in-haskell/)のような)。
たとえば、`take()`を使ってその一部を要求することができます。

```kotlin
println(fibonacci.take(10).joinToString())
```

> これは`1, 1, 2, 3, 5, 8, 13, 21, 34, 55`をプリントします。
  [ここ](examples/sequence/fibonacci.kt)でこのコードを試すことができます。

ジェネレータの長所は、`while`(上記の例から)、`if`、`try`/`catch`/`finally`などの任意の制御フローをサポートしているところです。

```kotlin
val seq = buildSequence {
    yield(firstItem) // 中断ポイント

    for (item in input) {
        if (!item.isValid()) break // これ以上アイテムを生成しない
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // 中断ポイント
    }

    try {
        yield(lastItem()) // 中断ポイント
    }
    finally {
        // 何らかの終了コード
    }
}
```

> `buildSequence{}`と `yield()`のライブラリコードは、
[制限付きサスペンド](#restricted-suspension)のセクションに示されています。

この方法では、ライブラリ関数(`buildSequence{}`と `yield()`も同様)で `yieldAll(sequence)` と表現することもできます。これは、遅延配列の結合を簡単にし、効率的な実装を可能にします。

### 非同期UI

一般的なUIアプリケーションには、すべてのUI操作が発生する単一のイベントディスパッチスレッドがあります。
他のスレッドからのUI状態の変更は通常許可されません。すべてのUIライブラリは、実行をUIスレッドに戻すためのプリミティブを提供します。たとえばSwingには [`SwingUtilities.invokeLater`](https://docs.oracle.com/javase/8/docs/api/javax/swing/SwingUtilities.html#invokeLater-java.lang.Runnable-)、JavaFXには [`Platform.runLater`](https://docs.oracle.com/javase/8/javafx/api/javafx/application/Platform.html#runLater-java.lang.Runnable-)、Androidには [`Activity.runOnUiThread`](https://developer.android.com/reference/android/app/Activity.html#runOnUiThread(java.lang.Runnable))など。
ここでは、いくつかの非同期操作を行い、その結果をUIに表示する典型的なSwingアプリケーションのコードスニペットを示します。

```kotlin
makeAsyncRequest {
    // このラムダは、非同期要求が完了したときに実行される
    result, exception ->

    if (exception == null) {
        // 結果をUIに表示
        SwingUtilities.invokeLater {
            display(result)
        }
    } else {
       // 例外処理
    }
}
```

これは、[非同期計算](#asynchronous-computations)ユースケースのコールバック地獄に似ていますが、これもコルーチンによってエレガントに解決されています。

```kotlin
launch(Swing) {
    try {
        // 非同期でリクエストを作成している間中断する
        val result = makeRequest()
        // 結果をUIに表示する、ここではSwingコンテキストは常にイベントディスパッチスレッドに留まることを保証する
        display(result)
    } catch (exception: Throwable) {
        // process exception
    }
}
```

> `Swing`コンテキストのライブラリコードは[継続インターセプター](#continuation-interceptor)セクションに示されています。

すべての例外処理は、自然な言語構造を使用して実行されます。

### その他のユースケース

コルーチンは、次のような多くのユースケースをカバーできます。

* チャネルベースの同時実行性（goroutinesとチャネルとも呼ばれます）
* アクターベースの同時実行性
* ユーザとの対話を必要とするバックグラウンドプロセス。たとえば、モーダルダイアログの表示
* 通信プロトコル: 各アクターを状態マシンではなくシーケンスとして実装する
* Webアプリケーションのワークフロー: ユーザーの登録、電子メールのバリデーション、ログイン
(サスペンドされたコルーチンを直列化してDBに格納することができます)

## コルーチンの概要

このセクションでは、コルーチンとそのセマンティクスを制御する標準ライブラリを記述するための言語メカニズムの概要を説明します。

### コルーチンの実験的状態

コルーチンはKotlin 1.1で実験的です。なぜなら、我々は設計が変わることを想定しているからです。Kotlinコンパイラは、コルーチン関連の機能の使用に関する警告を出します。警告を取り除く `-Xcoroutines=enable` オプトインスイッチがあります 。

kotlin-stdlibのコルーチンに関連するすべてのAPIは、`kotlin.coroutines.experimental` パッケージに入っています。最終的な設計では、`kotlin.coroutines' パッケージで公開されますが、実験的なパッケージはしばらくの間そのままなので、古いバイナリは互換性があり作業を続けられます。

パブリックAPIでコルーチンを使用するすべてのライブラリで同じことを行う必要があります。したがって、ここでライブラリを作成しておき、将来のバージョンのユーザーを気にかける場合は、パッケージに `org.my.library.experimental` のような名前を付ける必要があります 。
そして、コルーチンの最終的な設計が来たら、メインAPIから `experimental` サフィックスを削除しますが、バイナリ互換性が必要なユーザーのために古いパッケージを残しておきます。

> 詳細は、[この](https://discuss.kotlinlang.org/t/experimental-status-of-coroutines-in-1-1-and-related-compatibility-concerns/2236) フォーラムの投稿で見つけることができます

### 用語

* _コルーチン_ - _中断可能な計算_の_インスタンス_です。
  概念的にはスレッドに似ていて、実行するコードブロックを持ち、同様のライフサイクルを持ち、_作成_され_起動_されますが、特定のスレッドに束縛されていません。あるスレッドで実行を_中断_し、別のスレッドで_再開_することがあります。
  さらに、futureやpromiseのように、何らかの結果や例外で_完了_します。
 
* _サスペンド関数_ - `suspend` 修飾子でマークされた関数。
  他のサスペンド関数を呼び出すことによって、現在の実行スレッドをブロックせずにコードの実行を_中断_することができます。
  サスペンド関数は、通常のコードから呼び出すことはできませんが、他のサスペンド関数やサスペンドラムダからのみ呼び出すことができます（下記参照）。
  例えば、 [ユースケース](#ユースケース)で示されている `.await()` や `yield()` は、ライブラリに定義されているサスペンド関数です。
  標準ライブラリは、他のすべてのサスペンド関数を定義するために使用されるプリミティブなサスペンド関数を提供します。

* _サスペンドラムダ_ - コルーチンで実行できるコードブロック。
  通常の[ラムダ式](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/lambdas.html)と全く同じように見えますが、 その関数型は `suspend` 修飾子でマークされています。
  通常のラムダ式と同様に、匿名のローカル関数用の短い構文形式です。サスペンドラムダは、匿名のサスペンド関数の短い構文形式です。
  サスペンド関数を呼び出すことによって、現在の実行スレッドをブロックせずにコードの実行を_中断_することができます。
  例えば、 [ユースケース](#ユースケース)で示されている `launch`、` future`、`buildSequence` 関数に続く中括弧で囲まれたコードブロックは、サスペンドラムダです。

> 注：サスペンドラムダは、このラムダの[非ローカル](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/returns.html) `return` 文が許可されているコードのすべての場所でサスペンド関数を呼び出せます。
  つまり、 [`apply{}` ブロック](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html) のようなインラインラムダの中で関数呼び出しを中断することは許されますが、 `noinline` や `crossinline` の内部ラムダ式では許可されません。
  _中断_は特別な種類の非ローカル制御転送として扱われます。

* _サスペンド関数型_ - サスペンド関数とサスペンドラムダの関数型です。
  通常の[関数型](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/lambdas.html#関数型)に似ていますが、`suspend` 修飾子が付いています。
  たとえば、`suspend() -> Int` は `Int` を返す引数のないサスペンド関数の型です。
  `suspend fun foo(): Int` のように宣言されたサスペンド関数は、この関数型に準拠しています。

* _コルーチンビルダー_ - 引数としていくつかの_サスペンドラムダ_を取り、コルーチンを作成し、オプションでその結果へのアクセスを与える関数。
  たとえば、 ユースケースで示した `launch{}`、`future{}`、`buildSequence{}` は、ライブラリに定義されているコルーチンのビルダーです。
  標準ライブラリは、他のすべてのコルーチンビルダーを定義するためのプリミティブなコルーチンビルダーを提供します。

> 注：一部の言語では、実行と結果の表現方法を定義するコルーチンを作成して開始する特定の方法をハードコーディングでサポートしています。
たとえば、`generate` _キーワード_はある種の反復可能オブジェクトを返すコルーチンを定義し、` async` _キーワード_ はある種のpromiseやタスクを返すコルーチンを定義します。
Kotlinには、コルーチンを定義して開始するためのキーワードや修飾子はありません。
コルーチンのビルダーは、単にライブラリ内で定義された関数です。
コルーチンの定義が別の言語のメソッド本体の形を取っている場合、Kotlinでは典型的に、式本体を持つ通常のメソッドであり、最後の引数がサスペンドラムダであるライブラリ定義のコルーチンビルダーの呼び出しから成ります。
 
```kotlin
fun asyncTask() = async { ... }
```

* 中断ポイント_ - コルーチンの実行が_中断される可能性がある_ポイントです。
構文的には、中断ポイントはサスペンド関数の呼び出しですが、中断関数が標準ライブラリプリミティブを呼び出して実行を中断すると、実際の中断が発生します。

* _継続_ - 中断ポイントで中断したコルーチンの状態です。中断ポイントのあとの残りの実行を概念的に表しています。例えば、

```kotlin
buildSequence {
    for (i in 1..10) yield(i * i)
    println("over")
}  
```  

ここでは、コルーチンがサスペンド関数 `yield()` の呼び出しで中断されるたびに_残りの実行_は継続として表されるので、10個の継続があります。
最初にループを実行して `i = 2` で中断し、2番目にループを実行して `i = 3` で中断し、最後のコマンドで "over" を出力してコルーチンを完成させます。
_作成_されてまだ_開始_されていないコルーチンは、その実行全体からなる `Continuation<Unit>' 型の初期の継続によって表されます。

前述したように、コルーチンの駆動要件の1つは柔軟性です。既存の多くの非同期APIやその他のユースケースをサポートし、コンパイラーにハードコードされた部分を最小限に抑えたいと考えています。その結果コンパイラは、サスペンド関数、サスペンドラムダ、および対応するサスペンド関数型のサポートのみを行います。標準ライブラリにはいくつかのプリミティブがあり、残りはアプリケーションライブラリに残されています。

### 継続インタフェース

一般的なコールバックを表す標準ライブラリインタフェース `Continuation` の定義を次に示します。

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resume(value: T)
   fun resumeWithException(exception: Throwable)
}
```

コンテキストは、[コルーチンコンテキスト](#コルーチンコンテキスト)セクションの詳細でカバーされ、コルーチンに関連付けられている任意のユーザ定義コンテキストを表します。
関数 `resume` と ` resumeWithException` は、`resume` で成功した結果を提供するか、コルーチン完了時に `resumeWithException` で失敗を報告する完了コールバックです。

### サスペンド関数

`.await()` のような典型的な_サスペンド関数_の実装は次のようになります。
  
```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) // futureは正常に終了した
                cont.resume(result)
            else // futureは例外で終了した
                cont.resumeWithException(exception)
        }
    }
``` 

> [ここ](examples/future/await.kt)でこのコードを入手できます。注：この簡単な実装は、futureが完了しなければ、コルーチンを永久に中断します。実際の[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実装は取り消しをサポートするのでやや複雑です。

`suspend` 修飾子は、コルーチンの実行を中断することができる関数であることを表しています。
この特定の関数は、`CompletableFuture<T>` 型の[拡張関数](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/extensions.html)として定義され、実際の実行順序に対応して左から右の順に自然に読みます。

```kotlin
asyncOperation(...).await()
```
 
修飾子 `suspend` は、トップレベル関数、拡張関数、メンバー関数、または演算子関数のいずれかの関数で使用できます。

> 現在のリリースのローカル関数、プロパティのゲッター/セッターおよびコンストラクタには `suspend` 修飾子を付けることはできません。これらの規制は将来廃止される予定です。
 
サスペンド関数は通常の関数を呼び出すことができますが、実際に実行を中断するには他のサスペンド関数を呼び出す必要があります。
特に、この `await` 実装は、次の方法でトップレベルのサスペンド関数として標準ライブラリに定義されたサスペンド関数 ` suspendCoroutine` を呼び出します。

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

コルーチンの内部で `suspendCoroutine` が呼び出されると（これはサスペンド関数であるため、コルーチン内でのみ呼び出すことができます）、コルーチンの実行を_中断_し、_継続_インスタンスにその状態をキャプチャして、この継続を指定された`ブロック `に引数として渡します。
コルーチンの実行を再開するために、ブロックは他のスレッドでこのスレッドの `continuation.resume()` または `continuation.resumeWithException()` を呼び出すことができます。
同じ継続を2回以上再開することは許されず、`IllegalStateException` を生成します。

> 注：これは、Kotlinのコルーチンと、SchemeやHaskelの継続モナドのような関数言語のファーストクラスの限定継続との主要な違いです。
制限された一度だけの再開のみのサポートを選択することは純粋に実用的で、意図された[ユースケース](#ユースケース)がファーストクラスの継続を必要としないため、それらの限定バージョンをより効率的に実装することができます。
しかし、ファーストクラスの継続は、継続にキャプチャされたコルーチンの状態を複製することによって別のライブラリとして実装することができ、その複製を再び再開することができます。
このメカニズムは、将来、標準ライブラリによって効率的に提供される可能性があります。

`continuation.resume()` に渡された値は `suspendCoroutine()` の**戻り値**になり、コルーチンが実行を_再開_すると `.await()` の戻り値になります。

### コルーチンビルダー

サスペンド関数は通常の関数から呼び出すことができないので、標準ライブラリは通常の非サスペンドスコープからコルーチンの実行を開始する関数を提供します。シンプルな `launch{}` _コルーチンビルダー_の実装を以下に示します。

```kotlin
fun launch(context: CoroutineContext, block: suspend () -> Unit) =
        block.startCoroutine(StandaloneCoroutine(context))

private class StandaloneCoroutine(override val context: CoroutineContext): Continuation<Unit> {
    override fun resume(value: Unit) {}

    override fun resumeWithException(exception: Throwable) {
        val currentThread = Thread.currentThread()
        currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
    }
}
```

> [ここ](examples/run/launch.kt)でこのコードを入手できます。

この実装は、このコルーチンを表す単純なクラス `StandaloneCoroutine` を定義し、その完了を取得するための `Continuation` インタフェースを実装します。
コルーチンの完了はその_完了の継続_を呼び出します。
その `resume` または `resumeWithException` 関数は、コルーチンが結果または例外での_完了_に対応して呼び出されます。
`launch`はコルーチンを「撃ち放し」にするため、戻り値の型は `Unit` 型でサスペンド関数が定義され、実際には `resume` 関数でこの結果を無視します。
コルーチンの実行が例外で完了すると、現在のスレッドのキャッチされていない例外ハンドラがそれを報告するために使用されます。

> 注：この単純な実装は `Unit` を返し、コルーチンの状態へのアクセスをまったく提供しません。
[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実際の実装はより複雑です。なぜなら、コルーチンを表す `Job`インタフェースのインスタンスを返し、取り消すことができるからです。

コンテキストは、[コルーチンコンテキスト](#コルーチンコンテキスト)のセクションで詳しく説明されています。
ここでは、有用なコンテキスト要素を定義する可能性がある他のライブラリとのより良い_合成_のために、ライブラリ定義のコルーチンビルダーに `context` パラメータを含めることは良いスタイルだと言えば十分です。

`startCoroutine` は、スタンダードライブラリ内で、サスペンド関数型の拡張として定義されています。
そのシグネイチャーは次のとおりです。

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
```

`startCoroutine`はコルーチンを作成し、現在のスレッドですぐに最初の_中断ポイント_まで実行を開始し（以下の注釈を参照してください）、リターンします。
中断ポイントはコルーチン本体での[サスペンド関数](#サスペンド関数)の呼び出しで、コルーチンの実行がいつどのように再開するかは、対応するサスペンド関数のコードによって決まります。

> 注：後で説明する[継続インターセプター](#継続インターセプター)（コンテキストから）は、最初の継続を_含む_コルーチンの実行を別のスレッドにディスパッチできます。

### コルーチンコンテキスト

コルーチンコンテキストは、コルーチンに添付できる永続的なユーザ定義オブジェクトのセットです。これは、コルーチンのスレッドポリシー、ロギング、コルーチンの実行のセキュリティとトランザクションの様相、コルーチンの識別子と名前などを担当するオブジェクトを含むことができます。コルーチンとそのコンテキストの単純なメンタルモデルがあります。コルーチンを軽量スレッドと考えてください。この場合、コルーチンのコンテキストは、スレッドローカル変数の集合のようなものです。
相違点は、スレッドローカル変数は変更可能でコルーチンのコンテキストは不変であることですが、コンテキストの中で何かを変更する必要があるときに新しいコルーチンを起動するのは簡単で軽量であるため、コルーチンの重大な制限ではありません。

標準ライブラリには、コンテキスト要素の具体的な実装は含まれていませんが、インタフェースと抽象クラスがあり、これらのすべての様相をライブラリで_構成可能_な方法で定義することができるため、異なるライブラリからの様相を同じ文脈の要素として平和的に共存させることができます。

概念的には、コルーチンのコンテキストは、各要素に一意のキーを持つ要素のインデックス付きセットです。これは、セットとマップのミックスです。その要素はマップのようなキーを持っていますが、キーは要素に直接関連付けられています。標準ライブラリは、`CoroutineContext` のための最小限のインタフェースを定義します。

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?
    fun <R> fold(initial: R, operation: (R, Element) -> R): R
    operator fun plus(context: CoroutineContext): CoroutineContext
    fun minusKey(key: Key<*>): CoroutineContext

    interface Element : CoroutineContext {
        val key: Key<*>
    }

    interface Key<E : Element>
}
```

'CoroutineContext' はそれ自身に利用可能な4つの中核操作を持っています。

* `get` 演算子は、指定されたキーの要素への型セーフなアクセスを提供します。これは、[Kotlin演算子のオーバーロード](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/operator-overloading.html)で説明した[..]表記法で使用できます。
* `fold` 関数は 標準ライブラリの [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) 拡張のように機能し、コンテキスト内のすべての要素を反復する手段を提供します。
* 演算子 `plus` は 標準ライブラリの [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html) 拡張のように機能し、2つのコンテキストの組みの左側の要素を同じキーの右側の要素で置き換えて返します。
* 関数 `minusKey` は、指定されたキーを含まないコンテキストを返します。

コルーチンコンテキストの `Element` はコンテキスト自体です。 この要素のみを持つシングルトンコンテキストです。
これにより、コルーチンコンテキスト要素のライブラリ定義を取得し、それらを `+` で結合することによって、複合コンテキストを作成することができます。
たとえば、あるライブラリがユーザー認証情報を持つ `auth` 要素を定義し、いくつかの他のライブラリがいくつかの実行コンテキスト情報を持つ `CommonPool` オブジェクトを定義している場合、`launch（auth + CommonPool）{...}` を呼び出すことでコンテキストを組み合わせた `launch{}` [コルーチンビルダー](#コルーチンビルダー)を使うことができます。

> 注：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)は、コルーチンの実行をバックグラウンドスレッドの共有プールにディスパッチする `CommonPool` オブジェクトを含むいくつかのコンテキスト要素を提供します。

すべてのライブラリ定義のコンテキスト要素は、標準ライブラリによって提供される `AbstractCoroutineContextElement` クラスを継承するものとします。
ライブラリ定義のコンテキスト要素には、次のスタイルが推奨されます。
次の例は、現在のユーザー名を格納する仮の認可コンテキスト要素を示しています。

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

対応する要素クラスのコンパニオンオブジェクトとしてのコンテキスト `Key` の定義は、コンテキストの対応する要素への流暢なアクセスを可能にする。
現在のユーザーの名前を確認する必要があるサスペンド関数の仮説的実装です。

```kotlin
suspend fun secureAwait(): Unit = suspendCoroutine { cont ->
    val currentUser = cont.context[AuthUser]?.name
    // ユーザー固有の処理を行う 
}
```

### 継続インターセプター

[非同期UI](#非同期UI)のユースケースを要約しましょう。非同期UIアプリケーションでは、さまざまなサスペンド関数が任意のスレッドでコルーチンの実行を再開しても、コルーチン本体自体が常にUIスレッドで実行されるようにする必要があります。これは、_継続インターセプター_を使用して達成されます。まず、コルーチンのライフサイクルを完全に理解する必要があります。[`launch{}`](#コルーチンビルダー)コルーチンビルダーを使用したコードスニペットを考えてみましょう 。

```kotlin
launch(CommonPool) {
    initialCode() // 初期コードの実行
    f1.await() // 中断ポイント #1
    block1() // 実行 #1
    f2.await() // 中断ポイント #2
    block2() // 実行 #2
}
```

コルーチンは最初の中断ポイントまで `initialCode` の実行を開始します。
中断ポイントでは_中断_し、しばらくして対応するサスペンド関数で定義されたように `block1` の実行に_復帰_し、再び中断して `block2` の実行に復帰した後、_完了_します。

継続インターセプターには `initialCode`、` block1`、および `block2` の実行に対応する継続を、それらの後続の中断ポイントへの再開からインターセプトしてラップするオプションがあります。
コルーチンの初期コードは、_初期継続_の再開として扱われます。
標準ライブラリは、次のインタフェースを提供します。
 
```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
}
 ```
 
この `interceptContinuation` 関数は、コルーチンの継続をラップします。コルーチンが中断されると、コルーチンフレームワークは次のコード行を使用して、後続の再開のために実際の `continuation` をラップします。

```kotlin
val facade = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```
 
コルーチンフレームワークは、継続の各実際のインスタンスの結果としてのファサードをキャッシュします。詳細については、 [実装の詳細](#実装の詳細)のセクションを参照してください。

実行をSwing UIイベントディスパッチスレッドにディスパッチする `Swing` インターセプターの具体的なコード例を見てみましょう。
現在のスレッドをチェックし、継続がSwingイベントディスパッチスレッドでのみ再開することを確認する `SwingContinuation` ラッパークラスの定義から始めます。
実行がUIスレッドで既に発生している場合、 `Swing` はすぐに適切な `cont.resume`を呼び出します。そうでなければ  `SwingUtilities.invokeLater` を使用して継続の実行をSwing UIスレッドにディスパッチします。

```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> by cont {
    override fun resume(value: T) {
        if (SwingUtilities.isEventDispatchThread()) cont.resume(value)
        else SwingUtilities.invokeLater { cont.resume(value) }
    }

    override fun resumeWithException(exception: Throwable) {
        if (SwingUtilities.isEventDispatchThread()) cont.resumeWithException(exception)
        else SwingUtilities.invokeLater { cont.resumeWithException(exception) }
    }
}
```

次に、対応するコンテキスト要素として機能する `Swing` オブジェクトを定義し、` ContinuationInterceptor` インタフェースを実装します。
  
```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

> [ここ](examples/context/swing.kt)でこのコードを入手できます。
注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実際の `Swing` オブジェクトの実装は、現在このコルーチンを実行しているスレッドの名前で現在実行中のコルーチンの識別子を提供して表示するコルーチンデバッグ機能もサポートしています。

さて、完全にSwingイベントディスパッチスレッドで実行されているコルーチンを実行するために、 `launch{}` [コルーチンビルダー](#コルーチンビルダー)と `Swing` パラメータを使用することができます
 
```kotlin
launch(Swing) {
   // ここのコードは中断できますが、常にSwing EDTで再開します
}
```

### 制限付きサスペンド

[ジェネレーター](#ジェネレーター)のユースケースから `buildSequence{}` と `yield()` を実装するには、異なる種類のコルーチンビルダーとサスペンド関数が必要です。
以下は、`buildSequence{}` コルーチンビルダーのライブラリコードです。

```kotlin
fun <T> buildSequence(block: suspend SequenceBuilder<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

これは、コルーチンを_作成_する `createCoroutine` という標準ライブラリの異なるプリミティブを使用しますが、起動_しません_。
代わりに、`Continuation<Unit>` への参照として_最初の継続_を返します。
もう1つの違いは、このビルダーの_サスペンドラムダ_ `block` は `SequenceBuilder<T>` レシーバーを持つ[_拡張ラムダ_](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/lambdas.html#レシーバ付き関数リテラル)です。
`SequenceBuilder` インターフェースは、ジェネレーターブロックの_スコープ_を提供し、ライブラリとして次のように定義されています。

```kotlin
interface SequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

複数のオブジェクトの作成を避けるため、`buildSequence {}`実装は `SequenceBuilder<T>` を実装する `SequenceCoroutine <T>` クラスを定義し、`Continuation<Unit>` も実装していますので、`createCoroutine` の `receiver` パラメータ としても `completion` 継続パラメータとしても機能します。
`SequenceCoroutine<T>` の単純な実装を以下に示します。

```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceBuilder<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // AbstractIterator implementation
    override fun computeNext() { nextStep.resume(Unit) }

    // Completion continuation implementation
    override val context: CoroutineContext get() = EmptyCoroutineContext
    override fun resume(value: Unit) { done() }
    override fun resumeWithException(exception: Throwable) { throw exception }

    // Generator implementation
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```
 
> [ここ](examples/sequence/buildSequence.kt)でこのコードを入手できます

`yield`の実装は `suspendCoroutine` [サスペンド関数](#サスペンド関数)を使ってコルーチンを中断し、その継続を捕捉します。
`computeNext` が呼び出されると、継続は `nextStep` として保存され、再開されます。
 
しかし、上に示した `buildSequence{}` と `yield()` は、任意のサスペンド関数がそのスコープ内の継続を捕捉する準備ができていません。それらは_同期_して動作します。
継続をどのようにキャプチャし、どこに格納し、いつ再開するかを絶対的に制御する必要があります。
_制限付きサスペンションスコープ_を形成します。
サスペンションを制限する機能は、スコープクラスまたはインタフェースに配置された `@RestrictsSuspension` アノテーションによって提供されます。上記の例では、このスコープインタフェースは `SequenceBuilder` です。

```kotlin
@RestrictsSuspension
interface SequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

この注釈は、`SequenceBuilder{}` または同様の同期コルーチンビルダーのスコープで使用できるサスペンド関数に一定の制限を適用します。
_制限付きサスペンションスコープ_のクラスまたはインタフェース（`@RestrictsSuspension` でマークされている）をレシーバーとして持つサスペンドラムダまたはサスペンド関数の拡張は、_制限付きサスペンド関数_と呼ばれます。
制限付きサスペンド関数は、同じ制限付きサスペンションスコープのインスタンスの中でのみメンバ関数または拡張関数を呼び出すことができます。
具体的には、スコープ内のラムダの `SequenceBuilder` 拡張が `suspendContinuation` やその他の一般的なサスペンド関数を呼び出せないことを意味します。
`generate` コルーチンの実行を中断するには、最終的に `SequenceBuilder.yield` を呼び出さなければなりません。
`yield` 自体の実装は `Generator` 実装のメンバー関数であり、何の制約もありません（_拡張_サスペンドラムダと関数のみ制限があります）。

これは `sequenceBuilder` のような制限されたコルーチンビルダーのために任意のコンテキストをサポートすることはほとんど意味がないので、常に `EmptyCoroutineContext` で動作するようにハードコードされています。

## その他の例

これは、新しい言語構造やライブラリ関数を導入しない非規範的なセクションですが、すべてのビルディングブロックがどのようにさまざまなユースケースをカバーするように構成されているかを示しています。

### コールバックのラッピング

多くの非同期APIには、コールバックスタイルのインターフェイスがあります。
標準ライブラリの `suspendCoroutine`サスペンド関数は、コールバックをKotlinサスペンド関数に簡単にラップする方法を提供します。

単純なパターンがあります。
この計算の `Result` を受け取るコールバックを持つ `someLongComputation` 関数があるとします。

```kotlin
fun someLongComputation(params: Params, callback: (Result) -> Unit)
```

以下の簡単なコードを使用して、サスペンド関数に変換することができます。
 
```kotlin
suspend fun someLongComputation(params: Params): Result = suspendCoroutine { cont ->
    someLongComputation(params) { cont.resume(it) }
} 
```

現在、この計算の戻り値の型は明示的ですが、依然として非同期であり、スレッドをブロックしません。

もっと複雑な例として、[非同期計算](#非同期計算)のユースケースから `aRead()` 関数を見てみましょう。
これは、Java NIO [`AsynchronousFileChannel`](http://docs.oracle.com/javase/jp/8/docs/api/java/nio/channels/AsynchronousFileChannel.html) と [`CompletionHandler`](http://docs.oracle.com/javase/jp/8/docs/api/java/nio/channels/CompletionHandler.html) コールバックインタフェースのためのサスペンド拡張関数として次のコードで実装できます。

```kotlin
suspend fun AsynchronousFileChannel.aRead(buf: ByteBuffer): Int =
    suspendCoroutine { cont ->
        read(buf, 0L, Unit, object : CompletionHandler<Int, Unit> {
            override fun completed(bytesRead: Int, attachment: Unit) {
                cont.resume(bytesRead)
            }

            override fun failed(exception: Throwable, attachment: Unit) {
                cont.resumeWithException(exception)
            }
        })
    }
```

> [ここ](examples/io/io.kt)でこのコードを入手できます。
注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実際の実装では、長時間実行されるIO操作を中止するための取り消しがサポートされています。

すべて同じ型のコールバックを共有するたくさんの関数を扱っている場合は、共通のラッパー関数を定義してそれらをすべてサスペンド関数に簡単に変換することができます。
たとえば、[vert.x](http://vertx.io/)は、すべての非同期関数がコールバックとして `Handler<AsyncResult<T>` を受け取るという特定の規約を使用します。
コルーチンから簡単に任意のvert.x関数を使用するには、以下のヘルパー関数を定義することができます。

```kotlin
inline suspend fun <T> vx(crossinline callback: (Handler<AsyncResult<T>>) -> Unit) = 
    suspendCoroutine<T> { cont ->
        callback(Handler { result: AsyncResult<T> ->
            if (result.succeeded()) {
                cont.resume(result.result())
            } else {
                cont.resumeWithException(result.cause())
            }
        })
    }
```

このヘルパー関数を使うと、任意の非同期vert.x関数 `async.foo(params、handler)` を `vx { async.foo(params, it) }` でコルーチンから呼び出すことができます。

### futureの作成

[future](#futures)ユースケースの `future{}` ビルダーは、[コルーチンビルダー](#コルーチンビルダー)のセクションで説明したように、futureまたはpromiseプリミティブに対して `launch{}` ビルダーと同様に定義することができます。

```kotlin
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
        CompletableFutureCoroutine<T>(context).also { block.startCoroutine(completion = it) }
```

`launch{}` との最初の違いは [`CompletableFuture`](http://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/CompletableFuture.html) の実装を返すことです。もう一つの違いはデフォルトの `CommonPool` コンテキストで定義されているため、デフォルトの実行動作は [`ForkJoinPool.commonPool`](http://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--) でコードを実行する [`CompletableFuture.supplyAsync`](http://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) メソッドと似ています。
`CompletableFutureCoroutine` の基本的な実装は簡単です。

```kotlin
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
    override fun resume(value: T) { complete(value) }
    override fun resumeWithException(exception: Throwable) { completeExceptionally(exception) }
}
```

このコルーチンの完了は、このコルーチンの結果を記録するため、futureの対応する `complete` メソッドを呼び出します。

### ノンブロッキングスリープ

コルーチンでは、スレッドをブロックする [`Thread.sleep`](https://docs.oracle.com/javase/jp/8/docs/api/java/lang/Thread.html#sleep-long-) は使うべきではありません。
しかし、Javaの [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html) を使用して、ノンブロッキングの `delay` サスペンド関数を実装するのは非常に簡単です。

```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
    Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont ->
    executor.schedule({ cont.resume(Unit) }, time, unit)
}
```

> [ここ](examples/delay/delay.kt)でこのコードを入手できます。
注：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)は `delay` 関数も提供します。

この種の `delay` 関数は、単一の「スケジューラー」スレッドでコルーチンを再開することに注意してください。
`Swing` のような[インターセプター](#継続インターセプター)を使用しているコルーチンは、インターセプターが適切なスレッドにディスパッチするので、このスレッドでは実行されません。
インターセプターを持たないコルーチンは、このスケジューラーのスレッドで実行されたままになります。
このソリューションはデモの目的には便利ですが、最も効率的な方法ではありません。
対応するインターセプターでネイティブにスリープを実装することをお勧めします。

For `Swing` interceptor that native implementation of non-blocking sleep shall use
[Swing Timer](https://docs.oracle.com/javase/8/docs/api/javax/swing/Timer.html)
that is specifically designed for this purpose:

以下のためのSwing非ブロッキング睡眠のネイティブ実装を使用しなければならないことを迎撃 スイングタイマー この目的のために特別に設計されています：

`Swing` インターセプターの場合、ノンブロッキングスリープのネイティブ実装では、この目的のために特別に設計された[Swing Timer](https://docs.oracle.com/javase/jp/8/docs/api/javax/swing/Timer.html)を使用します。

```kotlin
suspend fun Swing.delay(millis: Int): Unit = suspendCoroutine { cont ->
    Timer(millis) { cont.resume(Unit) }.apply {
        isRepeats = false
        start()
    }
}
```

> [ここ](examples/context/swing-delay.kt)でこのコードを入手できます。
注：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の `delay`の実装は、インターセプター固有のスリープ機能を認識し、適切な場合には上記の方法を自動的に使用します。

### 協調シングルスレッドマルチタスキング

同時実行性と共有可能な可変状態を処理する必要がないので、協調型シングルスレッドアプリケーションを作成するのが非常に便利です。JS、Python、その他多くの言語はスレッドを持ちませんが、協調的なマルチタスキングプリミティブを持っています。

[コルーチンインターセプター](#コルーチンインターセプター)は、すべてのコルーチンが1つのスレッドに限定されるようにするための簡単なツールを提供します。
[ここ](examples/context/threadContext.kt)のサンプルコードでは、シングルスレッド実行サービスを作成し、それをコルーチンインターセプターの要件に適合させる `newSingleThreadContext()` 関数を定義しています。

以下は、内部にアクティブな2つの非同期タスクを持っているにもかかわらず単一のスレッドで動作する例です。[futureの作成](#futureの作成)セクションで定義された `future{}` コルーチンビルダーでこれを使用します。

```kotlin
fun main(args: Array<String>) {
    log("Starting MyEventThread")
    val context = newSingleThreadContext("MyEventThread")
    val f = future(context) {
        log("Hello, world!")
        val f1 = future(context) {
            log("f1 is sleeping")
            delay(1000) // sleep 1s
            log("f1 returns 1")
            1
        }
        val f2 = future(context) {
            log("f2 is sleeping")
            delay(1000) // sleep 1s
            log("f2 returns 2")
            2
        }
        log("I'll wait for both f1 and f2. It should take just a second!")
        val sum = f1.await() + f2.await()
        log("And the sum is $sum")
    }
    f.get()
    log("Terminated")
}
```

> [ここ](examples/context/threadContext-example.kt)で完全に動作する例を得ることができます。
注：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)には、`newSingleThreadContext` のすぐに使える実装があります。

アプリケーション全体がシングルスレッド実行に基づいている場合は、シングルスレッド実行機能用のハードコーディングされたコンテキストを持つ独自のヘルパーコルーチンビルダーを定義できます。
  
### 非同期シーケンス

[制限付きサスペンド](#制限付きサスペンド)セクションに示されている `buildSequence{}` コルーチンビルダーは、_同期_コルーチンの例です。
コルーチンのプロデューサーコードは、コンシューマーが `Iterator.next()` を呼び出すと同時に同じスレッドで同期的に呼び出されます。
`buildSequence{}` コルーチンブロックは制限されており、[コールバックのラッピング](#コールバックのラッピング)セクションに示すように、非同期ファイルIOのようなサードパーティのサスペンド関数を使って実行を中断することはできません。

_非同期_シーケンスビルダーは、その実行を任意に中断して再開することができます。
データがまだ作成されていない場合、そのコンシューマーはそのケースを処理する準備ができていることを意味します。
これはサスペンド関数の自然な使用例です。通常の [`Iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterator/)インタフェースに似た `SuspendingIterator` インタフェースを定義しましょう。ただし、`next()` と `hasNext()` 関数は中断できます。
 
```kotlin
interface SuspendingIterator<out T> {
    suspend operator fun hasNext(): Boolean
    suspend operator fun next(): T
}
```

[`Sequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence/index.html) と似ていますが、`SuspendingIterator` を返します。

```kotlin
interface SuspendingSequence<out T> {
    operator fun iterator(): SuspendingIterator<T>
}
```

また、同期シーケンスビルダーのスコープに似ているスコープインターフェイスを定義しますが、その中断は制限されていません。

```kotlin
interface SuspendingSequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

ビルダー関数 `suspendingSequence{}` は同期 `generate{}` に似ています。
それらの違いは、`SuspendingIteratorCoroutine` の実装の詳細と、この場合はオプションのコンテキストを受け入れることが理にかなっていることです。

```kotlin
fun <T> suspendingSequence(
        context: CoroutineContext = EmptyCoroutineContext,
        block: suspend SuspendingSequenceBuilder<T>.() -> Unit
): SuspendingSequence<T> = object : SuspendingSequence<T> {
    override fun iterator(): SuspendingIterator<T> = suspendingIterator(context, block)

}
```

> [ここ](examples/suspendingSequence/suspendingSequence.kt)で完全なコードを取得できます。
注：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)には、同じコンセプトのより柔軟な実装を提供する、`Channel`プリミティブと対応する `produce{}` コルーチンビルダーの実装があります。

[協調シングルスレッドマルチタスキング](#協調シングルスレッドマルチタスキング)セクションから `newSingleThreadContext{}` コンテキストを、[ノンブロッキングスリープ](#ノンブロッキングスリープ)セクションからノンブロッキング `delay` 関数を取り出しましょう。
このようにして、1から10までの整数が得られ、それらの間に500msのスリープがあるノンブロッキングシーケンスの実装を書くことができます。
 
```kotlin
val seq = suspendingSequence(context) {
    for (i in 1..10) {
        yield(i)
        delay(500L)
    }
}
```
   
コンシューマーコルーチンは、このシーケンスを独自のペースで消費することができ、他の任意のサスペンド関数も中断します。
注目すべきは、Kotlin [forループ](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/control-flow.html)は慣習的に動作するので、言語に特別な `await for` ループ構造は必要ないことです。
通常の `for` ループは、上で定義した非同期シーケンスを反復するために使用できます。
プロデューサーに値がない場合は中断されます。

```kotlin
for (value in seq) { //プロデューサーを待っている間に中断
    // ここで値を使って何かする。ここでも中断するかもしれない。
}
```

> [ここ](examples/suspendingSequence/suspendingSequence-example.kt)で、実行を説明するログ出力を伴う実際の例を見つけることができます。

### チャネル

Goスタイルのタイプセーフなチャンネルは、Kotlinでライブラリとして実装できます。サスペンド関数 `send` として送信チャネルのインタフェースを定義することができます。

```kotlin
interface SendChannel<T> {
    suspend fun send(value: T)
    fun close()
}
```
  
そして、[非同期シーケンス](#非同期シーケンス)と同様のスタイルでサスペンド関数 `receive` と `operator iterator` を持つレシーバーチャネル、

```kotlin
interface ReceiveChannel<T> {
    suspend fun receive(): T
    suspend operator fun iterator(): ReceiveIterator<T>
}
```

`Channel<T>` クラスは両方のインタフェースを実装しています。
`send` はチャネルバッファーがいっぱいになると中断し、`receive` はバッファーが空の時に中断します。
Go-styleコードをほぼそのままKotlinにコピーすることができます。
Goのツアーの4番目の並行処理の例の、`n` 個のフィボナッチ数をチャネルに送信する `fibonacci` 関数は、Kotlinでは次のようになります。

```kotlin
suspend fun fibonacci(n: Int, c: SendChannel<Int>) {
    var x = 0
    var y = 1
    for (i in 0..n - 1) {
        c.send(x)
        val next = x + y
        x = y
        y = next
    }
    c.close()
}

```

Goスタイルの `go {...}` ブロックを定義して、ある種のマルチスレッドプールで新しいコルーチンを開始することもできます。このコルーチンは、任意の数の軽量コルーチンを決まった数の実際の重いスレッドにディスパッチします。
[ここ](examples/channel/go.kt)の実装例は、Javaの一般的な `ForkJoinPool` の上に書かれています。

この `go` コルーチンビルダーを使用すると、対応するGoコードのmain関数は次のようになります。`mainBlocking` は `runBlocking` のためのショートカットヘルパー関数で、`go{}` と同じプールを使用します：

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val c = Channel<Int>(2)
    go { fibonacci(10, c) }
    for (i in c) {
        println(i)
    }
}
```

> [ここ](examples/channel/channel-example-4.kt)で動作するコードをチェックアウトすることができます

チャネルのバッファーサイズは自由にいじることができます。
簡単にするため、バッファーされていないチャネルは概念的に前に説明した[非同期シーケンス](#非同期シーケンス)と類似しているため、この例ではバッファされたチャネルのみが実装されています（最小バッファサイズ1）。

アクションの1つがチャンネルの1つで利用可能になるまで停止するGoスタイルの `select` 制御ブロックは、Kotlin DSLとして実装することができ、[Goのツアーの5番目の並行処理の例](https://tour.golang.org/concurrency/5)は、Kotlinでは次のようになります。
 
```kotlin
suspend fun fibonacci(c: SendChannel<Int>, quit: ReceiveChannel<Int>) {
    var x = 0
    var y = 1
    whileSelect {
        c.onSend(x) {
            val next = x + y
            x = y
            y = next
            true // whileループを継続する
        }
        quit.onReceive {
            println("quit")
            false // whileループを抜ける
        }
    }
}
```

> [ここ](examples/channel/channel-example-5.kt)で動作するコードをチェックアウトすることができます
  
例には、Kotlinの [`when` 式](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/control-flow.html#when式)のようにケースの1つを結果として返す `select {...}` と、中括弧の少ない `while(select<Boolean> { ... })` と同じ便利な `whileSelect { ... }` の両方の実装があります。
  
[Goのツアーの6番目の並行処理の例](https://tour.golang.org/concurrency/6)のデフォルトの選択例では、もう1つのケースが `select {...}` DSLに追加されます。

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val tick = Time.tick(100)
    val boom = Time.after(500)
    whileSelect {
        tick.onReceive {
            println("tick.")
            true // ループを継続する
        }
        boom.onReceive {
            println("BOOM!")
            false // ループを抜ける
        }
        onDefault {
            println("    .")
            delay(50)
            true // ループを継続する
        }
    }
}
```

> [ここ](examples/channel/channel-example-6.kt)で動作するコードをチェックアウトすることができます

`Time.tick` と `Time.after` は[ここ](examples/channel/time.kt)ではノンブロッキング `delay` 関数で簡単に実装されています。
  
その他の例は、コメント内の対応するGoコードへのリンクと共に[ここ](examples/channel/)にあります。

このチャネルの実装例は、内部待機リストを管理するための単一のロックに基づいていることに注意してください。これは、理解しやすく、理由を簡単にします。ただし、このロックの下でユーザーコードを実行することはないため、完全に同時です。このロックは、スケーラビリティを非常に多数の同時スレッドに制限します。

> [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)でのチャネルと `select` の実際の実装は、ロックフリーのdisjoint-access-parallelデータ構造に基づいています。

このチャネルの実装は、コルーチンのコンテキストのインターセプターとは独立しています。
これは、対応する継続インターセプターのセクションに示されているように、イベントスレッドインターセプターの下でUIアプリケーションで使用することも、他のインターセプターで使用することも、インターセプターをまったく使用しないこともできます（後者の場合、実行スレッドは、コルーチンで使用される他のサスペンド関数のコードによってのみ決定されます）。
チャネル実装は、スレッドセーフなノンブロッキングサスペンド関数を提供するだけです。

### ミューテックス

スケーラブルな非同期アプリケーションを書くことは、実際にスレッドがブロックされることなく、コードがブロックされずに中断される（中断機能を使用して）ことを確実にする規律です。
[`ReentrantLock`](https://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)のようなJava並行処理プリミティブはスレッドブロッキングであり、本当に非ブロッキングコードで使用すべきではありません。
共有リソースへのアクセスを制御するために、コルーチンをブロックするのではなく実行を中断する `Mutex`クラスを定義することができます。
対応するクラスのヘッダは次のようになります：

```kotlin
class Mutex {
    suspend fun lock()
    fun unlock()
}
```

> [ここ](examples/mutex/mutex.kt)で完全な実装を得ることができます。
[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実際の実装には、いくつかの追加機能があります。

この非ブロッキングミューテックスの実装を使用すると、[Goのツアーの9番目の並行処理の例](https://tour.golang.org/concurrency/9)は、Goの `defer` と同じ目的を果たすKotlinの [`try-finally`](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/exceptions.html) を使ってKotlinに変換できます。

```kotlin
class SafeCounter {
    private val v = mutableMapOf<String, Int>()
    private val mux = Mutex()

    suspend fun inc(key: String) {
        mux.lock()
        try { v[key] = v.getOrDefault(key, 0) + 1 }
        finally { mux.unlock() }
    }

    suspend fun get(key: String): Int? {
        mux.lock()
        return try { v[key] }
        finally { mux.unlock() }
    }
}
```

> [ここ](examples/channel/channel-example-9.kt)で動作するコードをチェックアウトすることができます

## Advanced topics

This section covers some advanced topics dealing with resource management, concurrency, 
and programming style.
このセクションでは、リソース管理、同時実行性、およびプログラミングスタイルに関するいくつかの高度なトピックについて説明します。

### Resource management and GC

Coroutines don't use any off-heap storage and do not consume any native resources by themselves, unless the code
that is running inside a coroutine does open a file or some other resource. While files opened in a coroutine must
be closed somehow, the coroutine itself does not need to be closed. When coroutine is suspended its whole state is 
available by the reference to its continuation. If you lose the reference to suspended coroutine's continuation,
then it will be ultimately collected by garbage collector.
Coroutineは、オフヒープストレージを使用せず、コルーチン内で実行されているコードがファイルやその他のリソースを開かない限り、それ自身ではネイティブリソースを消費しません。コルーチン内で開かれたファイルは何とか閉じなければなりませんが、コルーチン自体を閉じる必要はありません。コルーチンが中断されている場合、コルーチン全体の状態はその続行を参照することで利用可能です。中断されたコルーチンの継続に関する参照を失った場合、最終的にはガベージコレクタによって収集されます。

Coroutines that open some closeable resources deserve a special attention. Consider the following coroutine
that uses the `buildSequence{}` builder from [restricted suspension](#restricted-suspension) section to produce
a sequence of lines from a file:
閉鎖可能なリソースを開くコルーチンは、特別な注意が必要です。制限付きサスペンド・セクションからbuildSequence{}ビルダーを使用して、ファイルから一連の行を生成する次のコルーチンを考えてみましょう。

```kotlin
fun sequenceOfLines(fileName: String) = buildSequence<String> {
    BufferedReader(FileReader(fileName)).use {
        while (true) {
            yield(it.readLine() ?: break)
        }
    }
}
```

This function returns a `Sequence<String>` and you can use this function to print all lines from a file 
in a natural way:
この関数はa Sequence<String>を返し、この関数を使って自然な方法でファイルからすべての行を出力することができます：
 
```kotlin
sequenceOfLines("examples/sequence/sequenceOfLines.kt")
    .forEach(::println)
```

> You can get full code [here](examples/sequence/sequenceOfLines.kt)
ここで完全なコードを取得できます

It works as expected as long as you iterate the sequence returned by the `sequenceOfLines` function
completely. However, if you print just a few first lines from this file like here:
sequenceOfLines関数が返すシーケンスを完全に反復する限り、期待どおりに動作します。しかし、ここのようにこのファイルから最初の行を数行だけ印刷すると、

```kotlin
sequenceOfLines("examples/sequence/sequenceOfLines.kt")
        .take(3)
        .forEach(::println)
```

then the coroutine resumes a few times to yield the first three lines and becomes _abandoned_.
It is Ok for the coroutine itself to be abandoned but not for the open file. The 
[`use` function](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 
will not have a chance to finish its execution and close the file. The file will be left open
until collected by GC, because Java files have a `finalizer` that closes the file. It is 
not a big problem for a small slide-ware or a short-running utility, but it may be a disaster for
a large backend system with multi-gigabyte heap, that can run out of open file handles 
faster than it runs out of memory to trigger GC.
コルーチンは数回再開して最初の3行を生成し、放棄されます。コルーチン自体が放棄されていても、開いているファイルには残っていないのはOKです。この use関数 は、実行を終了してファイルを閉じる機会はありません。Javaファイルにはfinalizerファイルを閉じるため、GCによって収集されるまでファイルは開いたままになります。小さなスライドウェアや短時間実行のユーティリティでは大きな問題ではありませんが、マルチギガバイトのヒープを持つ大規模なバックエンドシステムでは、開いているファイルハンドルが使い果たされたときよりも速く実行される可能性がありますGCをトリガするメモリ。

This is a similar gotcha to Java's 
[`Files.lines`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#lines-java.nio.file.Path-)
method that produces a lazy stream of lines. It returns a closeable Java stream, but most stream operations do not
automatically invoke the corresponding 
`Stream.close` method and it is up to the user to remember about the need to close the corresponding stream. 
One can define closeable sequence generators 
in Kotlin, but they will suffer from a similar problem that no automatic mechanism in the language can
ensure that they are closed after use. It is explicitly out of the scope of Kotlin coroutines
to introduce a language mechanism for an automated resource management.
これは、JavaのFiles.lines メソッドに似 た問題であり、遅延のあるストリームを生成します。クローズ可能なJavaストリームを返しますが、ほとんどのストリーム操作では対応するStream.closeメソッドが自動的に起動されるわけではなく 、対応するストリームを閉じる必要性について覚えておく必要があります。Kotlinではクローズ可能なシーケンスジェネレータを定義できますが、言語上の自動メカニズムが使用後に閉じられることを保証できない同様の問題が発生します。自動化されたリソース管理のための言語メカニズムを導入することは、明示的にKotlinコルーチンの範囲外です。

However, usually this problem does not affect asynchronous use-cases of coroutines. An asynchronous coroutine 
is never abandoned, but ultimately runs until its completion, so if the code inside a coroutine properly closes
its resources, then they will be ultimately closed.
しかし、通常、この問題はコルーチンの非同期ユースケースには影響しません。非同期コルーチンは放棄されることはありませんが、最終的に完了するまで実行されるため、コルーチン内のコードが適切にリソースを閉じると、最終的にクローズされます。

### Concurrency and threads

Each individual coroutine, just like a thread, is executed sequentially. It means that the following kind 
of code is perfectly safe inside a coroutine:
個々のコルーチンは、スレッドと同様に、順次実行されます。これは、次のような種類のコードがコルーチン内で完全に安全であることを意味します。

```kotlin
launch(CommonPool) { // starts a coroutine // コルーチンを開始
    val m = mutableMapOf<String, String>()
    val v1 = someAsyncTask1().await() // suspends on await // awaitで中断
    m["k1"] = v1 // modify map when resumed // 再開したらマップを変更
    val v2 = someAsyncTask2().await() // suspends on await
    m["k2"] = v2 // modify map when resumed // 再開したらマップを変更
}
```

You can use all the regular single-threaded mutable structures inside the scope of a particular coroutine.
However, sharing mutable state _between_ coroutines is potentially dangerous. If you use a coroutine builder
that install a dispatcher to resume all coroutines JS-style in the single event-dispatch thread, 
like the `Swing` interceptor shown in [continuation interceptor](#continuation-interceptor) section,
then you can safely work with all shared
objects that are generally modified from this event-dispatch thread. 
However, if you work in multi-threaded environment or otherwise share mutable state between
coroutines running in different threads, then you have to use thread-safe (concurrent) data structures. 
特定のコルーチンのスコープ内で、すべての通常のシングルスレッドの可変構造を使用できます。しかし、コルーチン間で変更可能な状態を共有することは潜在的に危険です。あなたのような、単一のイベントディスパッチスレッドですべてのコルーチンのJS-スタイルを再開するためにディスパッチャをインストールコルーチンビルダーを使用する場合Swingに示される迎撃継続インターセプタセクションを、あなたが安全に、一般的に、このイベントから変更されたすべての共有オブジェクトを操作することができます - ディスパッチスレッド。ただし、マルチスレッド環境で作業している場合や、別のスレッドで実行されているコルーチン間で可変状態を共有している場合は、スレッドセーフ（並行）データ構造を使用する必要があります。

Coroutines are like threads, albeit they are more lightweight. You can have millions of coroutines running on 
just a few threads. The running coroutine is always executed in some thread. However, a _suspended_ coroutine
does not consume a thread and it is not bound to a thread in any way. The suspending function that resumes this
coroutine decides which thread the coroutine is resumed on by invoking `Continuation.resume` on this thread 
and coroutine's interceptor can override this decision and dispatch the coroutine's execution onto a different thread.
コルーチンはもっと軽量ですがスレッドのようなものです。ほんの数個のスレッドで数百万のコルーチンを動かすことができます。実行中のコルーチンは、常にあるスレッドで実行されます。しかし、サスペンドされているコルーチンはスレッドを消費せず、スレッドにバインドされません。このコルーチンを再開する一時停止機能Continuation.resumeは、このスレッドを呼び出してコルーチンが再開されたスレッドを決定し、コルーチンのインターセプタはこの決定をオーバーライドしてコルーチンの実行を別のスレッドにディスパッチできます。

## Asynchronous programming styles

There are different styles of asynchronous programming.
 
Callbacks were discussed in [asynchronous computations](#asynchronous-computations) section and are generally
the least convenient style that coroutines are designed to replace. Any callback-style API can be
wrapped into the corresponding suspending function as shown [here](#wrapping-callbacks). 

Let us recap. For example, assume that you start with a hypothetical _blocking_ `sendEmail` function 
with the following signature:

さまざまなスタイルの非同期プログラミングがあります。

コールバックは非同期計算セクションで議論され、一般に、コルーチンが置き換えるように設計されている最も簡単なスタイルです。コールバックスタイルのAPIは、ここに示すように、対応する一時停止関数にラップすることができます。

私たちを要約しましょう。たとえば、次のシグネチャで仮想的なブロッキング sendEmail関数を使い始めるとします。

```kotlin
fun sendEmail(emailArgs: EmailArgs): EmailResult
```

It blocks execution thread for potentially long time while it operates.

To make it non-blocking you can use, for example, error-first 
[node.js callback convention](https://www.tutorialspoint.com/nodejs/nodejs_callbacks_concept.htm)
to represent its non-blocking version in callback-style with the following signature:

動作中に実行スレッドを潜在的に長時間ブロックします。

非ブロッキングにするには、たとえば、エラーファーストの node.jsコールバック規約 を使用して、非ブロッキングバージョンをコールバックスタイルで次のシグネチャで表すことができます。

```kotlin
fun sendEmail(emailArgs: EmailArgs, callback: (Throwable?, EmailResult?) -> Unit)
```

However, coroutines enable other styles of asynchronous non-blocking programming. One of them
is async/await style that is built into many popular languages.
In Kotlin this style can be replicated by introducing `future{}` and `.await()` library functions
that were shown as a part of [futures](#futures) use-case section.
 
This style is signified by the convention to return some kind of future object from the function instead 
of taking a callback as a parameter. In this async-style the signature of `sendEmail` is going to look like this:

しかし、コルーチンは、他のスタイルの非同期ノンブロッキングプログラミングを可能にします。それらの1つは、多くの一般的な言語に組み込まれているasync / awaitスタイルです。Kotlinではこのスタイルは、導入して複製することができるfuture{}と.await()の一部として示された機能ライブラリ先物ユースケースセクションを。

このスタイルは、コールバックをパラメータとして使用するのではなく、関数から何らかの将来のオブジェクトを返すように条約によって示されています。この非同期スタイルでは、署名は次のsendEmailようになります。

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult>
```

As a matter of style, it is a good practice to add `Async` suffix to such method names, because their 
parameters are no different from a blocking version and it is quite easy to make a mistake of forgetting about
asynchronous nature of their operation. The function `sendEmailAsync` starts a _concurrent_ asynchronous operation 
and potentially brings with it all the pitfalls of concurrency. However, languages that promote this style of 
programming also typically have some kind of `await` primitive to bring the execution back into the sequence as needed. 

Kotlin's _native_ programming style is based on suspending functions. In this style, the signature of 
`sendEmail` looks naturally, without any mangling to its parameters or return type but with an additional
`suspend` modifier:

スタイルの問題として、Asyncそのメソッド名には接尾辞を追加することをお勧めします。それらのパラメータはブロッキング版と変わらず、操作の非同期性を忘れることは間違いありません。この関数sendEmailAsyncは並行非同期操作を開始し、潜在的に並行処理の落とし穴をもたらします。しかし、このプログラミングスタイルを促進するawait言語には、通常、実行を必要に応じてシーケンスに戻すためのプリミティブがあります。

Kotlinのネイティブプログラミングスタイルは、機能を一時停止することに基づいています。このスタイルでは、 sendEmailそのパラメータや戻り値の型に変更を加えることなく、suspend修飾子を追加して、自然に署名が生成され ます。

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult
```

The async and suspending styles can be easily converted into one another using the primitives that we've 
already seen. For example, `sendEmailAsync` can be implemented via suspending `sendEmail` using
[`future` coroutine builder](#building-futures):
非同期スタイルと中断スタイルは、既に見たプリミティブを使用して簡単に相互に変換できます。たとえば、コルーチンビルダーを使用して 中断sendEmailAsyncすることで実装できます。sendEmailfuture

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult> = future {
    sendEmail(emailArgs)
}
```

while suspending function `sendEmail` can be implemented via `sendEmailAsync` using
[`.await()` suspending function](#suspending-functions)
サスペンド機能sendEmailをsendEmailAsync使用してサスペンド機能を 実装することもでき.await()ます

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult = 
    sendEmailAsync(emailArgs).await()
```

So, in some sense, these two styles are equivalent and are both definitely superior to callback style in their
convenience. However, let us look deeper at a difference between `sendEmailAsync` and suspending `sendEmail`.

Let us compare how they **compose** first. Suspending functions can be composed just like normal functions:

したがって、ある意味では、これらの2つのスタイルは同等であり、どちらも便利なコールバックスタイルより優れています。しかし、私たちはとの間の差でより深く見てみましょうsendEmailAsyncと懸濁sendEmail。

最初にどのように構成するかを比較してみましょう。一時停止機能は、通常の機能と同様に構成することができます。

```kotlin
suspend fun largerBusinessProcess() {
    // a lot of code here, then somewhere inside //ここにたくさんのコードがあります。
    sendEmail(emailArgs)
    // something else goes on after that //その後に何かが続く 
}
```

The corresponding async-style functions compose in this way:
対応する非同期スタイルの関数は次のように構成されます。

```kotlin
fun largerBusinessProcessAsync() = future {
   // a lot of code here, then somewhere inside
   sendEmailAsync(emailArgs).await()
   // something else goes on after that //その後に何かが続く 
}
```

Observe, that async-style function composition is more verbose and _error prone_. 
If you omit `.await()` invocation in async-style
example,  the code still compiles and works, but it now does email sending process 
asynchronously or even _concurrently_ with the rest of a larger business process, 
thus potentially modifying some shared state and introducing some very hard to reproduce errors.
On the contrary, suspending functions are _sequential by default_.
With suspending functions, whenever you need any concurrency, you explicitly express it in the source code with 
some kind of `future{}` or a similar coroutine builder invocation.

Compare how these styles **scale** for a big project using many libraries. Suspending functions are
a light-weight language concept in Kotlin. All suspending functions are fully usable in any unrestricted Kotlin coroutine.
Async-style functions are framework-dependent. Every promises/futures framework must define its own `async`-like 
function that returns its own kind of promise/future class and its own `await`-like function, too.

Compare their **performance**. Suspending functions provide minimal overhead per invocation. 
You can checkout [implementation details](#implementation-details) section.
Async-style functions need to keep quite heavy promise/future abstraction in addition to all of that suspending machinery. 
Some future-like object instance must be always returned from async-style function invocation and it cannot be optimized away even 
if the function is very short and simple. Async-style is not well-suited for very fine-grained decomposition.

Compare their **interoperability** with JVM/JS code. Async-style functions are more interoperable with JVM/JS code that 
uses a matching type of future-like abstraction. In Java or JS they are just functions that return a corresponding
future-like object. Suspending functions look strange from any language that does not support 
[continuation-passing-style](#continuation-passing-style) natively.
However, you can see in the examples above how easy it is to convert any suspending function into an 
async-style function for any given promise/future framework. So, you can write suspending function in Kotlin just once, 
and then adapt it for interop with any style of promise/future with one line of code using an appropriate 
`future{}` coroutine builder function.

async-style関数の構成はより冗長でエラーが発生しやすいことに注意してください。.await()非同期スタイルの例で呼び出しを省略すると、コードはコンパイルされて動作しますが、電子メールはプロセスを非同期に、または大規模なビジネスプロセスの他の部分と同時に送信するため、共有状態を変更したり、エラー。逆に、一時停止機能はデフォルトでは順次実行されます。サスペンド機能では、並行処理が必要なときはいつでも、何らかのfuture{}コルーチン・ビルダー呼び出しを使ってソース・コードで明示的に表現します。

多くのライブラリを使用する大きなプロジェクトで、これらのスタイルのスケールをどのように比較するか。一時停止機能は、Kotlinの軽量言語の概念です。すべてのサスペンド機能は、無制限のKotlinコルーチンで完全に使用できます。非同期スタイルの関数はフレームワークに依存します。あらゆる約束/先物の枠組みasyncは、それ自身の種類の約束/将来の階級とそれ自身のawaitような機能を返すそれ自体のような機能を定義しなければならない。

パフォーマンスを比較する。サスペンド機能は、呼び出しごとに最小限のオーバーヘッドを提供します。あなたはチェックアウト実施の詳細セクションができます。非同期スタイルの関数は、その中断されているすべての機械に加えて、かなりの有望性/将来の抽象化を維持する必要があります。将来のようなオブジェクトインスタンスは、非同期スタイルの関数呼び出しから常に返されなければならず、関数が非常に短く単純な場合でも最適化できません。非同期スタイルは非常に細かい分解には適していません。

JVM / JSコードとの相互運用性を比較する。非同期スタイルの関数は、将来のような抽象化のタイプを使用するJVM / JSコードとの相互運用性が向上します。JavaやJSでは、それは対応する将来のようなオブジェクトを返す関数に過ぎません。一時停止機能は、継続継承スタイルをネイティブにサポートしない言語からは変わってい ます。しかし、上の例では、任意の約束/将来のフレームワークに対して、サスペンド関数を非同期スタイルの関数に変換するのが簡単であることがわかります。したがって、Kotlinに一時停止機能を一度書くだけで、適切なfuture{}コルーチンビルダー機能を使用して、コードの1行で将来のあらゆるスタイルのinteropに適応させることができ ます。

## Implementation details

This section provides a glimpse into implementation details of coroutines. They are hidden
behind the building blocks explained in [coroutines overview](#coroutines-overview) section and
their internal classes and code generation strategies are subject to change at any time as
long as they don't break contracts of public APIs and ABIs.
このセクションでは、コルーチンの実装の詳細を垣間見ることができます。コルーチンの概要セクションで説明されているビルディングブロックの背後に隠されています。内部クラスやコード生成戦略は、公開APIやABIの契約を破らない限り、いつでも変更されることがあります。

### Continuation passing style

Suspending functions are implemented via Continuation-Passing-Style (CPS).
Every suspending function and suspending lambda has an additional `Continuation` 
parameter that is implicitly passed to it when it is invoked. Recall, that a declaration
of [`await` suspending function](#suspending-functions) looks like this:
サスペンド機能は、Continuation-Passing-Style（CPS）によって実装されます。すべての中断関数とラムダを中断するContinuation パラメータは、呼び出されるときに暗黙的に渡される追加パラメータを持ちます。await呼び出されると、関数の中断宣言は次のようになります。

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

However, its actual _implementation_ has the following signature after _CPS transformation_:
ただし、実際の実装では、CPS変換後に次のシグネチャがあります。

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

Its result type `T` has moved into a position of type argument in its additional continuation parameter.
The implementation result type of `Any?` is designed to represent the action of the suspending function.
When suspending function _suspends_ coroutine, it returns a special marker value of 
`COROUTINE_SUSPENDED`. When a suspending function does not suspend coroutine but
continues coroutine execution, it returns its result or throws an exception directly.
This way, the `Any?` return type of the `await` implementation is actually a union of
`COROUTINE_SUSPENDED` and `T` that cannot be expressed in Kotlin's type system.

The actual implementation of the suspending function is not allowed to invoke the continuation in its stack frame directly 
because that may lead to stack overflow on long-running coroutines. The `suspendCoroutine` function in
the standard library hides this complexity from an application developer by tracking invocations
of continuations and ensures conformance to the actual implementation contract of 
the suspending functions regardless of how and when the continuation is invoked.

その結果の型Tは、追加の継続パラメータで型引数の位置に移動しました。実装結果の型Any?は、中断機能の動作を表すように設計されています。サスペンド機能がコルーチンを中断すると、特殊マーカー値が返され COROUTINE_SUSPENDEDます。サスペンド機能がコルーチンを中断しないが、コルーチンの実行を続けると、その結果を返したり、例外を直接スローしたりします。このように、Any?の戻り値の型awaitの実装は、実際の労働組合である COROUTINE_SUSPENDEDとT、それはKotlinの型システムで表現することはできません。

一時停止関数の実際の実装では、スタックフレーム内の継続を直接呼び出すことはできません。これは、長期実行のコルーチンでスタックオーバーフローが発生する可能性があるためです。suspendCoroutine標準ライブラリの関数は、継続の呼び出しを追跡することによって、アプリケーション開発者からこの複雑さを隠し、関係なく、いつどのように継続が呼び出されるのサスペンド機能の実際の実装契約への適合性を保証します。

### State machines

It is crucial to implement coroutines efficiently, i.e. create as few classes and objects as possible.
Many languages implement them through _state machines_ and Kotlin does the same. In the case of Kotlin 
this approach results in the compiler creating only one class per suspending lambda that may
have an arbitrary number of suspension points in its body.   
 
Main idea: a suspending function is compiled to a state machine, where states correspond to suspension points. 
Example: let's take a suspending block with two suspension points:

コルーチンを効率的に実装すること、つまりできるだけクラスとオブジェクトを少なくすることが重要です。多くの言語がステートマシンを通じてそれらを実装し、Kotlinも同じことをしています。Kotlinの場合、このアプローチは、コンパイラが、体内に任意の数のサスペンドポイントを持つ可能性のあるラムダを一時停止するクラスを1つだけ作成することになります。

主なアイデア：サスペンド機能は、サスペンドポイントに対応するステートマシンにコンパイルされます。例：2つのサスペンドポイントを持つサスペンドブロックを作成しましょう。
 
```kotlin
val a = a()
val y = foo(a).await() // suspension point #1
b()
val z = bar(a, y).await() // suspension point #2
c(z)
``` 

There are three states for this block of code:
 
 * initial (before any suspension point)
 * after the first suspension point
 * after the second suspension point
 
Every state is an entry point to one of the continuations of this block 
(the initial continuation continues from the very first line). 
 
The code is compiled to an anonymous class that has a method implementing the state machine, 
a field holding the current state of the state machine, and fields for local variables of 
the coroutine that are shared between states (there may also be fields for the closure of 
the coroutine, but in this case it is empty). Here's pseudo-Java code for the block above
that uses continuation passing style for invocation of suspending functions `await`:

このコードブロックには3つの状態があります。

* イニシャル（サスペンドポイントの前）
* 最初のサスペンド後
* 第2の懸架点の後

すべての状態は、このブロックの継続の1つのエントリポイントです（最初の継続は最初の行から続きます）。

このコードは、ステートマシンを実装するメソッド、ステートマシンの現在のステートを保持するフィールド、ステート間で共有されるコルーチンのローカル変数のフィールドを持つ匿名クラスにコンパイルされます（クロージャのフィールドこの場合コルーチンは空です）。一時停止関数の呼び出しのために継続継承スタイルを使用する、上記のブロックの擬似Javaコードはawait次のとおりです。
  
``` java
class <anonymous_for_state_machine> extends CoroutineImpl<...> implements Continuation<Object> {
    // The current state of the state machine//ステートマシンの現在の状態
    int label = 0
    
    // local variables of the coroutine
    //コルーチンのローカル変数
    A a = null
    Y y = null
    
    void resume(Object data) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        // data is expected to be `null` at this invocation
        //この呼び出しでdataは`null`になると予想されます
        a = a()
        label = 1
        data = foo(a).await(this) // 'this' is passed as a continuation 
        if (data == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L1:
        // external code has resumed this coroutine passing the result of .await() as data 
        y = (Y) data
        b()
        label = 2
        data = bar(a, y).await(this) // 'this' is passed as a continuation
        if (data == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L2:
        // external code has resumed this coroutine passing the result of .await() as data 
        Z z = (Z) data
        c(z)
        label = -1 // No more steps are allowed
        return
    }          
}    
```  

Note that there is a `goto` operator and labels because the example depicts what happens in the 
byte code, not in the source code.

Now, when the coroutine is started, we call its `resume()` — `label` is `0`, 
and we jump to `L0`, then we do some work, set the `label` to the next state — `1`, call `.await()`
and return if the execution of the coroutine was suspended. 
When we want to continue the execution, we call `resume()` again, and now it proceeds right to 
`L1`, does some work, sets the state to `2`, calls `.await()` and again returns in case of suspension.
Next time it continues from `L3` setting the state to `-1` which means 
"over, no more work to do". 

A suspension point inside a loop generates only one state, 
because loops also work through (conditional) `goto`:

gotoこの例は、ソースコードではなくバイトコードで何が起こるかを示しているため、演算子とラベルがあることに注意してください。

コルーチンが起動されると、私たちはresume()- labelと呼んで、0ジャンプしてL0、次に仕事labelをし、次の状態に設定し 、コルーチンの実行が中断された場合に1呼び出し.await()て返します。私たちは執行を続けたいときには、resume()もう一度電話をかけて、今すぐ右に進み L1、仕事をして、州を設定し2、電話をかけ.await()、中断した場合には再び戻ります。次回L3は、状態を設定することから、-1「終わったら、それ以上はやらない」という意味になります。

ループ内のサスペンドポイントは1つの状態のみを生成します。これはループも機能します（条件付き）goto。
 
```kotlin
var x = 0
while (x < 10) {
    x += nextNumber().await()
}
```

is generated as

``` java
class <anonymous_for_state_machine> extends CoroutineImpl<...> implements Continuation<Object> {
    // The current state of the state machine//ステートマシンの現在の状態
    int label = 0
    
    // local variables of the coroutine//コルーチンのローカル変数
    int x
    
    void resume(Object data) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        
      L0:
        x = 0
      LOOP:
        if (x > 10) goto END
        label = 1
        data = nextNumber().await(this) // 'this' is passed as a continuation // 'this'は継続として渡された
        if (data == COROUTINE_SUSPENDED) return // return if await had suspended execution//待つ場合に返す実行中断していた
      L1:
        // external code has resumed this coroutine passing the result of .await() as data 
        //外部コードをデータとして）.await（の結果を渡し、このコルーチンを再開しました 
        x += ((Integer) data).intValue()
        label = -1
        goto LOOP
      END:
        label = -1 // No more steps are allowed //これ以上のステップは許されません
        return 
    }          
}    
```  

### Compiling suspending functions

The compiled code for suspending function depends on how and when it invokes other suspending functions.
In the simplest case, a suspending function invokes other suspending functions only at _tail positions_ 
making _tail calls_ to them. This is a typical case for suspending functions that implement low-level synchronization 
primitives or wrap callbacks, as shown in [suspending functions](#suspending-functions) and
[wrapping callbacks](#wrapping-callbacks) sections. These functions invoke some other suspending function
like `suspendCoroutine` at tail position. They are compiled just like regular non-suspending functions, with 
the only exception that the implicit continuation parameter they've got from [CPS transformation](#continuation-passing-style)
is passed to the next suspending function in tail call.

> Note: in the current implementation `Unit`-returning function must include an explicit `return` statement 
with the invocation of the other suspending function in order for it to be recognized as a tail call.

In a case when suspending invocations appear in non-tail positions, the compiler creates a 
[state machine](#state-machines) for the corresponding suspending function. An instance of the state machine
object in created when suspending function is invoked and is discarded when it completes.

> Note: in the future versions this compilation strategy may be optimized to create an instance of a state machine 
only at first suspension point.

This state machine object, in turn, serves as the _completion continuation_ for the invocation of other
suspending functions in non-tail positions. This state machine object instance is updated and reused when 
the function makes multiple invocations to other suspending functions. 
Compare this to other [asynchronous programming styles](#asynchronous-programming-styles),
where each subsequent step of asynchronous processing is typically implemented with a separate, freshly allocated,
closure object.

関数を一時停止するためにコンパイルされたコードは、他の一時停止関数をいつどのように呼び出すかによって異なります。最も単純なケースでは、サスペンド機能のみで、他の懸濁関数を呼び出すテール位置 することテール呼び出し、それらに。サスペンド関数や 折り返しコールバックの節に示すように、低レベルの同期プリミティブやラップコールバックを実装する関数を一時停止する典型的なケースです。これらの関数はsuspendCoroutine、末尾にあるような他の中断関数を呼び出します。これらは、通常の非停止機能と同様にコンパイルされますが、唯一の例外はCPS変換から得た暗黙の継続パラメータが 終了呼び出しの次の一時停止機能に渡されることです。

> 注：現在の実装では、 - 戻り関数は、テールコールとして認識されるように、他の中断関数の呼び出しとともにUnit明示的なreturnステートメントを含める必要があります。

サスペンド呼び出しが非テール位置に現れる場合、コンパイラは、対応するサスペンド機能用のステートマシンを作成し ます。中断された関数が呼び出されたときに作成され、完了すると破棄されるステートマシンオブジェクトのインスタンス。

> 注：将来のバージョンでは、このコンパイル戦略を最適化して、最初の中断ポイントでのみステートマシンのインスタンスを作成することができます。

このステートマシンオブジェクトは、今度は、として機能完了継続非尾位置における他の懸濁化機能の起動のために。このステートマシンオブジェクトインスタンスは、関数が他の中断関数に複数の呼び出しを行うときに更新され、再利用されます。これを他の非同期プログラミングスタイルと比較します。非同期処理の後続の各ステップでは、通常、新しく割り当てられた独立したクロージャオブジェクトを実装します。

### Coroutine intrinsics

The actual implementation of `suspendCoroutine` suspending function in the standard library is written in Kotlin
itself and its source code is available as part of the standard library sources package. In order to provide for the
safe and problem-free use of coroutines, it wraps the actual continuation of the state machine 
into an additional object on each suspension of coroutine. This is perfectly fine for truly asynchronous use cases
like [asynchronous computations](#asynchronous-computations) and [futures](#futures), since the runtime costs of the 
corresponding asynchronous primitives far outweigh the cost of an additional allocated object. However, for
the [generators](#generators) use case this additional cost is prohibitive.

The `kotlin.coroutines.experimental.intrinsics` package in the standard library contains the function named `suspendCoroutineOrReturn`
with the following signature:

suspendCoroutine標準ライブラリに一時停止機能の実際の実装はKotlin自身で書かれており、そのソースコードは標準ライブラリソースパッケージの一部として利用可能です。コルーチンの安全で問題のない使用を提供するために、コルーチンの各サスペンション上の追加のオブジェクトにステートマシンの実際の継続をラップします。これは、対応する非同期プリミティブのランタイムコストが追加の割り当てオブジェクトのコストをはるかに上回るため、非同期計算や先物のような真の非同期ユースケースにとってはまったく問題ありません。しかし、発電機の使用例では、この追加コストは法外なものです。

kotlin.coroutines.experimental.intrinsics標準ライブラリのパッケージには、という名前の関数が含まれsuspendCoroutineOrReturn 、次の署名とを：

```kotlin
suspend fun <T> suspendCoroutineOrReturn(block: (Continuation<T>) -> Any?): T
```

It provides direct access to [continuation passing style](#continuation-passing-style) of suspending functions
and _unchecked_ reference to continuation. The user of 
`suspendCoroutineOrReturn` bears full responsibility of following CPS result convention, but gains slightly
better performance as a result. This convention is usually easy to follow for `buildSequence`/`yield`-like coroutines,
but attempts to write asynchronous `await`-like suspending functions on top of `suspendCoroutineOrReturn` are
**discouraged** as they are **extremely tricky** to implement correctly without the help of `suspendCoroutine`
and errors in these implementation attempts are typically [heisenbugs](https://en.wikipedia.org/wiki/Heisenbug)
that defy attempts to find and reproduce them via tests.
 
There are also functions called `createCoroutineUnchecked` with the following signatures:

サスペンド機能の継承パッシングスタイルへの直接アクセスと、継続への未確認の参照を提供します。ユーザーは、 suspendCoroutineOrReturnCPSの結果規約に従う責任を全うしますが、結果としてパフォーマンスが少し向上します。この規則は、のために従うことは、通常は簡単ですbuildSequence/ yield様コルーチンが、非同期書き込みを試みるawaitの上に機能サスペンド様suspendCoroutineOrReturnされている 落胆をそのまま非常にトリッキーなの助けを借りずに正しく実装するsuspendCoroutine これらの実装の試みで、エラー、典型的には特異なバグ 挑みますテストで見つけて再生しようとします。

次のシグネチャで呼び出されるcreateCoroutineUnchecked関数もあります。

```kotlin
fun <T> (suspend () -> T).createCoroutineUnchecked(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutineUnchecked(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

They return unchecked reference to the initial continuation (without an additional wrapper object).  
Optimization version of `buildSequence` via `createCoroutineUnchecked` is shown below:
それらは初期の継続への未チェックの参照を返します（追加のラッパーオブジェクトなし）。
最適化バージョンbuildSequenceを経由しては、createCoroutineUnchecked以下の通りであります：

```kotlin
fun <T> buildSequence(block: suspend SequenceBuilder<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutineUnchecked(receiver = this, completion = this)
    }
}
```

Optimized version of `yield` via `suspendCoroutineOrReturn` is shown below.
Note, that because `yield` always suspends, 
the corresponding block always returns `COROUTINE_SUSPENDED`.
最適化バージョンyieldを経由してはsuspendCoroutineOrReturn以下の通りです。そのため、注意してくださいyield必ず一時停止し、対応するブロックは常に返しますCOROUTINE_SUSPENDED。

```kotlin
// Generator implementation
override suspend fun yield(value: T) {
    setNext(value)
    return suspendCoroutineOrReturn { cont ->
        nextStep = cont
        COROUTINE_SUSPENDED
    }
}
```

> You can get full code [here](examples/sequence/buildSequenceOptimized.kt)

The contents of `kotlin.coroutines.experimental.intrinsics` package are hidden from auto-completion in Kotlin 
plugin for IDEA to protect them from accidental usage. You need to manually write the corresponding 
import statement to get access to the above intrinsics.

> ここで完全なコードを取得できます

kotlin.coroutines.experimental.intrinsicsパッケージの内容は、誤って使用されるのを防ぐため、IDEAのKotlinプラグインの自動補完から隠されています。上記の組み込み関数にアクセスするには、対応するimport文を手動で記述する必要があります。

 
## Revision history

This section gives an overview of changes between various revisions of coroutines design.

### Changes in revision 3.2

* Added description of `createCoroutineUnchecked` intrinsic.

### Changes in revision 3.1

This revision is implemented in Kotlin 1.1.0 release.

* `kotlin.coroutines` package is replaced with `kotlin.coroutines.experimental`.
* `SUSPENDED_MARKER` is renamed to `COROUTINE_SUSPENDED`.
* Clarification on experimental status of coroutines added.

### Changes in revision 3

This revision is implemented in Kotlin 1.1-Beta.

* Suspending functions can invoke other suspending function at arbitrary points.
* Coroutine dispatchers are generalized to coroutine contexts:
  * `CoroutineContext` interface is introduced.
  * `ContinuationDispatcher` interface is replaced with `ContinuationInterceptor`.
  * `createCoroutine`/`startCoroutine` parameter `dispatcher` is removed.
  * `Continuation` interface includes `val context: CoroutineContext`.
* `CoroutineIntrinsics` object is replaced with `kotlin.coroutines.intrinsics` package.

### Changes in revision 2

This revision is implemented in Kotlin 1.1-M04.

* The `coroutine` keyword is replaced by suspending functional type.
* `Continuation` for suspending functions is implicit both on call site and on declaration site.
* `suspendContinuation` is provided to capture continuation is suspending functions when needed.
* Continuation passing style transformation has provision to prevent stack growth on non-suspending invocations.
* `createCoroutine`/`startCoroutine` coroutine builders are introduced.
* The concept of coroutine controller is dropped:
  * Coroutine completion result is delivered via `Continuation` interface.
  * Coroutine scope is optionally available via coroutine `receiver`.
  * Suspending functions can be defined at top-level without receiver.
* `CoroutineIntrinsics` object contains low-level primitives for cases where performance is more important than safety.
 

