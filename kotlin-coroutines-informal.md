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

> `Swing`コンテキストのライブラリコードは[継続インターセプタ](#continuation-interceptor)セクションに示されています。

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

> 注：後で説明する[継続インターセプタ](#継続インターセプター)（コンテキストから）は、最初の継続を_含む_コルーチンの実行を別のスレッドにディスパッチできます。

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

An `Element` of the coroutine context is a context itself. It is a singleton context with this element only.
This enables creation of composite contexts by taking library definitions of coroutine context elements and
joining them with `+`. For example, if one library defines `auth` element with user authorization information,
and some other library defines `CommonPool` object with some execution context information,
then you can use a `launch{}` [coroutine builder](#coroutine-builders) with the combined context using
`launch(auth + CommonPool) {...}` invocation.
Elementコルーチンのコンテキストのコンテキストそのものです。この要素のみを持つシングルトンコンテキストです。これにより、コルーチンのコンテキスト要素のライブラリ定義を取得して結合することで、複合コンテキストを作成することができます+。たとえば、あるライブラリauthがユーザー認可情報を持つ要素を定義し、他の一部のライブラリがCommonPoolオブジェクトをいくつかの実行コンテキスト情報で定義している場合は、launch{} 呼び出しを使用して合成コンテキストでコルーチン・ビルダーを使用 launch(auth + CommonPool) {...}できます。

> Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) provides several context elements, 
  including `CommonPool` object that dispatches execution of coroutine onto a shared pool of background threads.
注意：kotlinx.coroutinesは、コルーチンの実行CommonPoolをバックグラウンドスレッドの共有プールにディスパッチするオブジェクトを含むいくつかのコンテキスト要素を提供します。

All library-defined context elements shall extend `AbstractCoroutineContextElement` class that is provided
by the standard library. The following style is recommended for library defined context elements.
The example below shows a hypothetical authorization context element that stores current user name:
すべてのライブラリ定義コンテキスト要素はAbstractCoroutineContextElement、標準ライブラリによって提供されるクラスを拡張するものとする。ライブラリ定義のコンテキスト要素には、次のスタイルが推奨されます。次の例は、現在のユーザー名を格納する仮の認可コンテキスト要素を示しています。

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

The definition of context `Key` as a companion object of the corresponding element class enables fluent access
to the corresponding element of the context. Here is a hypothetical implementation of suspending function that
needs to check the name of the current user:
Key対応する要素クラスの付随オブジェクトとしてのコンテキストの定義は、コンテキストの対応する要素への流暢なアクセスを可能にする。現在のユーザーの名前を確認する必要がある一時停止機能の仮説的実装です：

```kotlin
suspend fun secureAwait(): Unit = suspendCoroutine { cont ->
    val currentUser = cont.context[AuthUser]?.name
    // do something user-specific
    // ユーザー固有の操作を行う 
}
```

### Continuation interceptor 継続インターセプタ

Let's recap [asynchronous UI](#asynchronous-ui) use case. Asynchronous UI applications must ensure that the 
coroutine body itself is always executed in UI thread, despite the fact that various suspending functions 
resume coroutine execution in arbitrary threads. This is accomplished using a _continuation interceptor_.
First of all, we need to fully understand the lifecycle of a coroutine. Consider a snippet of code that uses 
[`launch{}`](#coroutine-builders) coroutine builder:
非同期UIの使用例を要約しましょう。非同期UIアプリケーションでは、さまざまな中断関数が任意のスレッドでコルーチンの実行を再開しても、コルーチン本体自体が常にUIスレッドで実行されるようにする必要があります。これは、継続インターセプタを使用して達成されます。まず、コルーチンのライフサイクルを完全に理解する必要があります。launch{}コルーチンビルダーを使用したコードスニペットを考えてみましょう 。

```kotlin
launch(CommonPool) {
    initialCode() // execution of initial code 初期コードの実行
    f1.await() // suspension point #1 中断ポイント #1
    block1() // execution #1 実行　#1
    f2.await() // suspension point #2 中断ポイント #2
    block2() // execution #2 実行 #2
}
```

Coroutine starts with execution of its `initialCode` until the first suspension point. At the suspension point it
_suspends_ and, after some time, as defined by the corresponding suspending function, it _resumes_ to execute 
`block1`, then it suspends again and resumes to execute `block2`, after which it _completes_.
コルーチンはinitialCode、最初の停止点まで実行を開始します。サスペンション時点で、 中断対応懸濁関数によって定義されるいくつかの時間の後、それは、と再開を実行するために block1、それは再び中断して実行するように再開しblock2、それはその後、完了する。

Continuation interceptor has an option to intercept and wrap the continuation that corresponds to the
execution of `initialCode`, `block1`, and `block2` from their resumption to the subsequent suspension points.
The initial code of the coroutine is treated as a
resumption of its _initial continuation_. The standard library provides the following interface:
継続インターセプタはインターセプトとの実行に対応する継続をラップするためのオプションがありinitialCode、block1およびblock2その後のサスペンションポイントへの再開からを。コルーチンの初期コードは、その再開のように扱われている初期継続。標準ライブラリは、次のインタフェースを提供します。
 
```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
}
 ```
 
The `interceptContinuation` function wraps the continuation of the coroutine. Whenever coroutine is suspended,
coroutine framework uses the following line of code to wrap the actual `continuation` for the subsequent
resumption:
このinterceptContinuation関数は、コルーチンの継続をラップします。コルーチンが中断されると、coroutineフレームワークは次のコード行を使用して、後続の再開のために実際のcontinuationものをラップします。

```kotlin
val facade = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```
 
Coroutine framework caches the resulting facade for each actual instance of continuation. See
[implementation details](#implementation-details) section for more details.
コルーチンのフレームワークは、継続の各実際のインスタンスの結果としてのファサードをキャッシュします。詳細については、 実装の詳細のセクションを参照してください。

Let us take a look at a concrete example code for `Swing` interceptor that dispatches execution onto
Swing UI event dispatch thread. We start with a definition of a `SwingContinuation` wrapper class that
checks the current thread and makes sure that continuation resumes only in Swing event dispatch thread.
If the execution already happens in UI thread, then `Swing` just invokes an appropriate `cont.resume` right away,
otherwise it dispatches execution of the continuation onto Swing UI thread using `SwingUtilities.invokeLater`.
SwingSwing UIイベントディスパッチスレッドに実行をディスパッチするインターセプタの具体的なコード例を見てみましょう。SwingContinuation現在のスレッドをチェックし、継続がスウィングイベントディスパッチスレッドでのみ再開するようにするラッパークラスの定義から始めます。実行がUIスレッドで既に発生している場合Swingは、すぐにcont.resume適切なメソッドを呼び出すだけです。それ以外の場合は、UIメソッドを使用して継続の実行をSwing UIスレッドにディスパッチしSwingUtilities.invokeLaterます。

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

Then define `Swing` object that is going to serve as the corresponding context element and implement
`ContinuationInterceptor` interface:
次にSwing、対応するコンテキスト要素として機能するオブジェクトを定義し、インタフェースを ContinuationInterceptor実装します。
  
```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

> You can get this code [here](examples/context/swing.kt).
  Note: the actual implementation of `Swing` object in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  also supports coroutine debugging facilities that provide and display the identifier of the currently running 
  coroutine in the name of the thread that is currently running this coroutine.
ここでこのコードを入手できます。注意：kotlinx.coroutinesのSwingオブジェクトの実際の実装で は、現在実行中のコルーチンの識別子を提供し、現在このコルーチンを実行しているスレッドの名前で表示するコルーチンデバッグ機能もサポートされています。

Now, one can use `launch{}` [coroutine builder](#coroutine-builders) with `Swing` parameter to 
execute a coroutine that is running completely in Swing event dispatch thread:
今では、launch{} Coroutine builderをSwingパラメータとともに使用して、Swingイベントディスパッチスレッドで完全に実行されているコルーチンを実行できます。
 
 ```kotlin
launch(Swing) {
   // code in here can suspend, but will always resume in Swing EDT
   // ここのコードは中断できますが、常にSwing EDTで再開します 
}
```

### Restricted suspension 制限付きサスペンション

A different kind of coroutine builder and suspension function is needed to implement `buildSequence{}` and `yield()`
from [generators](#generators) use case. Here is the library code for `buildSequence{}` coroutine builder:
実装するためにbuildSequence{}、そしてジェネレータのユースケースyield() から、異なる種類のコルーチンビルダーとサスペンション機能が必要です。コルーチンビルダーのライブラリコードは次のとおりです：buildSequence{}

```kotlin
fun <T> buildSequence(block: suspend SequenceBuilder<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

It uses a different primitive from the standard library called `createCoroutine` that _creates_ coroutine, 
but does _not_ start it. Instead, it returns its _initial continuation_ as a reference to `Continuation<Unit>`. 
The other difference is that _suspending lambda_
`block` for this builder is an 
[_extension lambda_](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver) 
with `SequenceBuilder<T>` receiver.
The `SequenceBuilder` interface provides the _scope_ for the generator block and is defined in a library as:
これは、コルーチンcreateCoroutineを作成するが、起動しない標準ライブラリとは異なるプリミティブを使用します。代わりに、最初の継続を参照先として返しますContinuation<Unit>。もう1つの違いは、このビルダーのラムダ blockをサスペンドすることは 、レシーバを持つ 拡張ラムダでSequenceBuilder<T>あることです。SequenceBuilderインタフェースは、提供範囲を発電ブロックのために、ライブラリなどで定義されています。

```kotlin
interface SequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

To avoid creation of multiple objects, `buildSequence{}` implementation defines `SequenceCoroutine<T>` class that
implements `SequenceBuilder<T>` and also implements `Continuation<Unit>`, so it can serve both as
a `receiver` parameter for `createCoroutine` and as its `completion` continuation parameter. 
The simple implementation for `SequenceCoroutine<T>` is shown below:
複数のオブジェクトの作成を避けるため、buildSequence{}インプリメンテーションはSequenceCoroutine<T>インプリメンテーションSequenceBuilder<T>とインプリメントも行うクラスを定義Continuation<Unit>するため、receiver継続パラメータのパラメータcreateCoroutineとしてもcompletion継続パラメータとしても機能します。の簡単な実装を以下SequenceCoroutine<T>に示します：

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
 
> You can get this code [here](examples/sequence/buildSequence.kt)
ここでこのコードを入手できます

The implementation of `yield` uses `suspendCoroutine` [suspending function](#suspending-functions) to suspend
the coroutine and to capture its continuation. Continuation is stored as `nextStep` to be resumed when the 
`computeNext` is invoked.
コルーチンを中断し、その継続を捕捉するための機能停止をyield使用する。継続が呼び出されると、継続が再開されるように 格納されます。suspendCoroutine nextStepcomputeNext
 
However, `buildSequence{}` and `yield()`, as shown above, are not ready for an arbitrary suspending function
to capture the continuation in their scope. They work _synchronously_.
They need absolute control on how continuation is captured, 
where it is stored, and when it is resumed. They form _restricted suspension scope_. 
The ability to restrict suspensions is provided by `@RestrictsSuspension` annotation that is placed
on the scope class or interface, in the above example this scope interface is `SequenceBuilder`:
しかし、buildSequence{}上yield()で示したように、任意の中断機能がその範囲内で継続をキャプチャする準備ができていません。それらは同期して動作します。継続をどのようにキャプチャし、どこに格納し、いつ再開するかを絶対的に制御する必要があります。制限されたサスペンションスコープを形成します。サスペンドを制限する機能@RestrictsSuspensionは、スコープクラスまたはインターフェイスに配置された注釈によって提供されます。上の例では、このスコープインターフェイスはSequenceBuilder次のとおりです。

```kotlin
@RestrictsSuspension
interface SequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

This annotation enforces certain restrictions on suspending functions that can be used in the
scope of `SequenceBuilder{}` or similar synchronous coroutine builder.
Any extension suspending lambda or function that has _restricted suspension scope_ class or interface 
(marked with `@RestrictsSuspension`) as its receiver is 
called a _restricted suspending function_.
Restricted suspending functions can only invoke member or
extension suspending functions on the same instance of their restricted suspension scope. 
In particular, it means that
no `SequenceBuilder` extension of lambda in its scope can invoke `suspendContinuation` or other
general suspending function. To suspend the execution of a `generate` coroutine they must ultimately invoke
`SequenceBuilder.yield`. The implementation of `yield` itself is a member function of `Generator`
implementation and it does not have any restrictions (only _extension_ suspending lambdas and functions are restricted).
この注釈では、SequenceBuilder{}同期コルーチン構築ツールのスコープまたは同様のコルーチン構築ツールで使用できる一時停止機能に一定の制限が適用されます。制限されたサスペンススコープクラスまたはインターフェイス（マークされている@RestrictsSuspension）を受信者として持つラムダまたは関数を中断する拡張子は、制限された中断関数と呼ばれます。制限された中断機能は、制限された中断範囲の同じインスタンスでメンバ機能または拡張機能を呼び出すことしかできません。特にSequenceBuilder、そのスコープ内のラムダの拡張が呼び出せないsuspendContinuation、または他の一般的な中断機能が呼び出せないことを意味します。コルーチンの実行を中断するには、最終的にgenerate呼び出さなければなりません SequenceBuilder.yield。yieldGenerator

It makes little sense to support arbitrary contexts for such a restricted coroutine builder as `sequenceBuilder`
so it is hardcoded to always work with `EmptyCoroutineContext`.
そのような限定されたコルーチンビルダーのために任意のコンテキストをサポートするのはほとんど意味がないsequenceBuilder ので、常に動作するようにハードコードされていEmptyCoroutineContextます。

## More examples

This is a non-normative section that does not introduce any new language constructs or 
library functions, but shows how all the building blocks compose to cover a large variety
of use-cases.
これは、新しい言語構造やライブラリ関数を導入しない非規範的なセクションですが、すべてのビルディングブロックがどのようにさまざまなユースケースをカバーするように構成されているかを示しています。

### Wrapping callbacks

Many asynchronous APIs have callback-style interfaces. The `suspendCoroutine` suspending function 
from the standard library provides for an easy way to wrap any callback into a Kotlin suspending function. 
多くの非同期APIには、コールバックスタイルのインターフェイスがあります。suspendCoroutine標準ライブラリから吊り機能はKotlinサスペンド機能に任意のコールバックをラップするための簡単な方法を提供します。

There is a simple pattern. Assume that you have `someLongComputation` function with callback that 
receives `Result` of this computation.
単純なパターンがあります。この計算someLongComputationを受け取るコールバックを持つ関数があると仮定してくださいResult。

```kotlin
fun someLongComputation(params: Params, callback: (Result) -> Unit)
```

You can convert it into a suspending function with the following straightforward code:
以下の簡単なコードを使用して、サスペンド機能に変換することができます。
 
```kotlin
suspend fun someLongComputation(params: Params): Result = suspendCoroutine { cont ->
    someLongComputation(params) { cont.resume(it) }
} 
```

Now the return type of this computation is explicit, but it is still asynchronous and does not block a thread.
現在、この計算の戻り値の型は明示的ですが、依然として非同期であり、スレッドをブロックしません。

For a more complex example let us take a look at
`aRead()` function from [asynchronous computations](#asynchronous-computations) use case. 
It can be implemented as a suspending extension function for Java NIO 
[`AsynchronousFileChannel`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/AsynchronousFileChannel.html)
and its 
[`CompletionHandler`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/CompletionHandler.html)
callback interface with the following code:
もっと複雑な例aRead()として、非同期計算のユースケースから関数を見てみましょう 。これは 、次のコードを使用して、Java NIO AsynchronousFileChannel およびその CompletionHandlerコールバックインタフェースのサスペンド拡張機能として実装 できます。

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

> You can get this code [here](examples/io/io.kt).
  Note: the actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)
  supports cancellation to abort long-running IO operations.
ここでこのコードを入手できます。注：内の実際の実装kotlinx.coroutinesは 長時間実行IO操作を中止するキャンセルをサポートしています。

If you are dealing with lots of functions that all share the same type of callback, then you can define a common
wrapper function to easily convert all of them to suspending functions. For example, 
[vert.x](http://vertx.io/) uses a particular convention that all its asynchronous functions receive 
`Handler<AsyncResult<T>>` as a callback. To simply the use of arbitrary vert.x functions from coroutines
the following helper function can be defined:
すべて同じ型のコールバックを共有するたくさんの関数を扱っている場合は、共通のラッパー関数を定義してそれらをすべて一時停止機能に簡単に変換することができます。たとえば、 vert.xでは、すべての非同期Handler<AsyncResult<T>>関数がコールバックとして受け取るという特定の規約が 使用されています。コルーチンからの任意のvert.x関数の単純な使用には、以下のヘルパー関数を定義することができます。

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

Using this helper function, an arbitrary asynchronous vert.x function `async.foo(params, handler)`
can be invoked from a coroutine with `vx { async.foo(params, it) }`.
このヘルパー関数を使用すると、任意の非同期vert.x関数async.foo(params, handler) をコルーチンから呼び出すことができvx { async.foo(params, it) }ます。

### Building futures

The `future{}` builder from [futures](#futures) use-case can be defined for any future or promise primitive
similarly to the `launch{}` builder as explained in [coroutine builders](#coroutine-builders) section:
future{}ビルダー先物はユースケースは、任意の将来のために定義されるか、または同様にプリミティブを約束することが可能launch{}で説明したようにビルダーコルーチンビルダーのセクション：

```kotlin
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
        CompletableFutureCoroutine<T>(context).also { block.startCoroutine(completion = it) }
```

The first difference from `launch{}` is that it returns an implementation of
[`CompletableFuture`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), 
and the other difference is that it is defined with a default `CommonPool` context, so that its default
execution behaviour is similar to the 
[`CompletableFuture.supplyAsync`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-)
method that runs its code in 
[`ForkJoinPool.commonPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--).
The basic implementation of `CompletableFutureCoroutine` is straightforward:
最初の相違点launch{}は、それが実装を返すこと CompletableFutureと、それがデフォルトのCommonPoolコンテキストで定義されて、そのデフォルトのCompletableFuture.supplyAsync 実行動作がコードを実行するメソッドに似ている こと ForkJoinPool.commonPoolです。の基本的な実装CompletableFutureCoroutineは簡単です：

```kotlin
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
    override fun resume(value: T) { complete(value) }
    override fun resumeWithException(exception: Throwable) { completeExceptionally(exception) }
}
```

> You can get this code [here](examples/future/future.kt).
  The actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) is more advanced,
  because it propagates the cancellation of the resulting future to cancel the coroutine.
ここでこのコードを入手できます。で実際の実装kotlinx.coroutinesは、それがコルーチンをキャンセルする結果として、将来の取り消しを伝播するため、より高度です。

The completion of this coroutine invokes the corresponding `complete` methods of the future to record the
result of this coroutine.
このコルーチンの完成は、このコルーチンのcomplete結果を記録する、将来の対応する方法を呼び出す。

### Non-blocking sleep

Coroutines should not use [`Thread.sleep`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-),
because it blocks a thread. However, it is quite straightforward to implement a suspending non-blocking `delay` function by using
Java's [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)
コルーチンはThread.sleepスレッドをブロックするので使用しないでください。ただし、delayJavaの関数を使用して非ブロッキング関数を一時停止することは、非常に簡単です。ScheduledThreadPoolExecutor

```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
    Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont ->
    executor.schedule({ cont.resume(Unit) }, time, unit)
}
```

> You can get this code [here](examples/delay/delay.kt).
  Node: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) also provides `delay` function.
ここでこのコードを入手できます。ノード：kotlinx.coroutinesはdelay関数も提供します。

Note, that this kind of `delay` function resumes the coroutines that are using it in its single "scheduler" thread.
The coroutines that are using [interceptor](#continuation-interceptor) like `Swing` will not stay to execute in this thread,
as their interceptor dispatches them into an appropriate thread. Coroutines without interceptor will stay to execute
in this scheduler thread. So this solution is convenient for demo purposes, but it is not the most efficient one. It
is advisable to implement sleep natively in the corresponding interceptors.
この種のdelay関数は、単一の「スケジューラー」スレッドでそれを使用しているコルーチンを再開することに注意してください。インターセプタのようなものSwingを使用しているコルーチンは、インターセプタが適切なスレッドにディスパッチするので、このスレッドでは実行されません。インターセプタを持たないコルーチンは、このスケジューラのスレッドで実行されたままになります。このソリューションはデモの目的には便利ですが、最も効率的な方法ではありません。対応するインターセプタでネイティブにスリープを実装することをお勧めします。

For `Swing` interceptor that native implementation of non-blocking sleep shall use
[Swing Timer](https://docs.oracle.com/javase/8/docs/api/javax/swing/Timer.html)
that is specifically designed for this purpose:
以下のためのSwing非ブロッキング睡眠のネイティブ実装を使用しなければならないことを迎撃 スイングタイマー この目的のために特別に設計されています：

```kotlin
suspend fun Swing.delay(millis: Int): Unit = suspendCoroutine { cont ->
    Timer(millis) { cont.resume(Unit) }.apply {
        isRepeats = false
        start()
    }
}
```

> You can get this code [here](examples/context/swing-delay.kt).
  Node: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) implementation of `delay` is aware of
  interceptor-specific sleep facilities and automatically uses the above approach where appropriate. 
ここでこのコードを入手できます。ノード：kotlinx.coroutinesの実装はdelay、インターセプタ固有のスリープ機能を認識し、適切な場合には上記の方法を自動的に使用します。

### Cooperative single-thread multitasking

It is very convenient to write cooperative single-threaded applications, because you don't have to 
deal with concurrency and shared mutable state. JS, Python and many other languages do 
not have threads, but have cooperative multitasking primitives.
同時実行性と共有可能な可変状態を処理する必要がないので、協調型シングルスレッドアプリケーションを作成するのが非常に便利です。JS、Python、その他多くの言語はスレッドを持ちませんが、協調的なマルチタスキングプリミティブを持っています。

[Coroutine interceptor](#coroutine-interceptor) provides a straightforward tool to ensure that
all coroutines are confined to a single thread. The example code
[here](examples/context/threadContext.kt) defines `newSingleThreadContext()` function that
creates a single-threaded execution services and adapts it to the coroutine interceptor
requirements.
Coroutineインターセプタは、すべてのコルーチンが1つのスレッドに限定されるようにするための簡単なツールを提供します。ここのサンプルコード ではnewSingleThreadContext()、シングルスレッド実行サービスを作成し、それをコルーチンインターセプタの要件に適合させる関数を定義しています。

We will use it with `future{}` coroutine builder that was defined in [building futures](#building-futures) section
in the following example that works in a single thread, despite the
fact that it has two asynchronous tasks inside that are both active.
両方のアクティブな内部に2つの非同期タスクが存在するにもかかわらず、単一のスレッドで動作する次の例の先物セクションをfuture{}作成する際に定義されたコルーチン構築ツールで使用します。

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

> You can get fully working example [here](examples/context/threadContext-example.kt).
  Node: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) has ready-to-use implementation of
  `newSingleThreadContext`. 
ここで完全に動作する例を得ることができます。ノード：kotlinx.coroutinesにはすぐに使用できる実装があり newSingleThreadContextます。

If your whole application is based on a single-threaded execution, you can define your own helper coroutine
builders with a hardcoded context for your single-threaded execution facilities.
アプリケーション全体がシングルスレッド実行に基づいている場合は、シングルスレッド実行機能用のハードコーディングされたコンテキストを持つ独自のヘルパーコルーチンビルダーを定義できます。
  
## Asynchronous sequences

The `buildSequence{}` coroutine builder that is shown in [restricted suspension](#restricted-suspension)
section is an example of a _synchronous_ coroutine. Its producer code in the coroutine is invoked
synchronously in the same thread as soon as its consumer invokes `Iterator.next()`. 
The `buildSequence{}` coroutine block is restricted and it cannot suspend its execution using 3rd-party suspending
functions like asynchronous file IO as shown in [wrapping callbacks](#wrapping-callbacks) section.
buildSequence{}示されているコルーチンビルダー制限サスペンション セクションは一例である同期コルーチン。コルーチンのプロデューサコードは、コンシューマが呼び出すと同時に同じスレッドで同期的に呼び出されますIterator.next()。buildSequence{}コルーチンブロックが制限され、に示すように、非同期ファイルIOのようなサードパーティ製吊り機能を使用してその実行を一時停止することはできませんコールバックのラッピングセクションを。

An _asynchronous_ sequence builder is allowed to arbitrarily suspend and resume its execution. It means
that its consumer shall be ready to handle the case, when the data is not produced yet. This is
a natural use-case for suspending functions. Let us define `SuspendingIterator` interface that is
similar to a regular 
[`Iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterator/) 
interface, but its `next()` and `hasNext()` functions are suspending:
非同期シーケンスビルダーは、任意に、その実行を中断し、再開することが許可されています。データがまだ作成されていない場合、その消費者はそのケースを処理する準備ができていることを意味します。これは機能を一時停止するための自然な使用例です。SuspendingIterator通常のIterator インターフェースに似たインターフェースを 定義しましょうが、そのインターフェースnext()とhasNext()機能は一時停止しています：
 
```kotlin
interface SuspendingIterator<out T> {
    suspend operator fun hasNext(): Boolean
    suspend operator fun next(): T
}
```

The definition of `SuspendingSequence` is similar to the standard
[`Sequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence/index.html)
but it returns `SuspendingIterator`:
定義はSuspendingSequence標準と似ています Sequence が、次のようになりますSuspendingIterator。

```kotlin
interface SuspendingSequence<out T> {
    operator fun iterator(): SuspendingIterator<T>
}
```

We also define a scope interface for that is similar to a scope of a synchronous sequence builder,
but it is not restricted in its suspensions:
また、同期シーケンスビルダのスコープに似ているスコープインターフェイスを定義しますが、その中断は制限されていません。

```kotlin
interface SuspendingSequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

The builder function `suspendingSequence{}` is similar to a synchronous `generate{}`.
Their differences lie in implementation details of `SuspendingIteratorCoroutine` and
in the fact that it makes sense to accept an optional context in this case:
ビルダー関数suspendingSequence{}は、同期関数に似ていgenerate{}ます。それらの違いは実装の詳細とSuspendingIteratorCoroutine、この場合はオプションのコンテキストを受け入れることが理にかなっています

```kotlin
fun <T> suspendingSequence(
        context: CoroutineContext = EmptyCoroutineContext,
        block: suspend SuspendingSequenceBuilder<T>.() -> Unit
): SuspendingSequence<T> = object : SuspendingSequence<T> {
    override fun iterator(): SuspendingIterator<T> = suspendingIterator(context, block)

}
```

> You can get full code [here](examples/suspendingSequence/suspendingSequence.kt).
  Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) has an implementation of
  `Channel` primitive with the corresponding `produce{}` coroutine builder that provides more 
  flexible implementation of the same concept.
ここでフルコードを取得できます。注：kotlinx.coroutinesは 、同じコンセプトのより柔軟な実装を提供Channelする対応するproduce{}コルーチンビルダーとのプリミティブの実装を備えています。

Let us take `newSingleThreadContext{}` context from
[cooperative single-thread multitasking](#cooperative-single-thread-multitasking) section
and non-blocking `delay` function from [non-blocking sleep](#non-blocking-sleep) section.
This way we can write an implementation of a non-blocking sequence that yields
integers from 1 to 10, sleeping 500 ms between them:
協調的なシングルスレッドのマルチタスクセクションとノンブロッキングスリープセクションからノンブロッキング機能へのnewSingleThreadContext{}コンテキストを 取ってみましょう。このようにして、非ブロッキングシーケンスの実装を書くことができます.1から10までの整数が得られ、それらの間に500msのスリープがあります。delay
 
```kotlin
val seq = suspendingSequence(context) {
    for (i in 1..10) {
        yield(i)
        delay(500L)
    }
}
```
   
Now the consumer coroutine can consume this sequence at its own pace, while also 
suspending with other arbitrary suspending functions. Note, that 
Kotlin [for loops](https://kotlinlang.org/docs/reference/control-flow.html#for-loops)
work by convention, so there is no need for a special `await for` loop construct in the language.
The regular `for` loop can be used to iterate over an asynchronous sequence that we've defined
above. It is suspended whenever producer does not have a value:
現在、消費者コルーチンは、このシーケンスを独自のペースで消費することができ、他の任意の中断機能も中断します。ループの ためのKotlin は慣例によって動作するのでawait for、言語の中で特別なループ構造は必要ありません。正規forループは、上で定義した非同期シーケンスを反復処理するために使用できます。プロデューサに値がない場合は一時停止されます。


```kotlin
for (value in seq) { // suspend while waiting for producer //プロデューサーを待っている間に一時停止
    // do something with value here, may suspend here, too
    //すぎて、ここで停止することができ、ここでの値で何かを行います 
}
```

> You can find a worked out example with some logging that illustrates the execution
  [here](examples/suspendingSequence-example.kt)
ここでの実行を説明するいくつかのログを使って、実際の例を見つけることができ ます

### Channels

Go-style type-safe channels can be implemented in Kotlin as a library. We can define an interface for 
send channel with suspending function `send`:
Goスタイルのタイプセーフなチャンネルは、Kotlinでライブラリとして実装できます。サスペンド機能付きの送信チャネルのインタフェースを定義することができますsend。

```kotlin
interface SendChannel<T> {
    suspend fun send(value: T)
    fun close()
}
```
  
and receiver channel with suspending function `receive` and an `operator iterator` in a similar style 
to [asynchronous sequences](#asynchronous-sequences):
サスペンド機能receiveを備えoperator iteratorたレシーバチャネル、および非同期シーケンスと同様のスタイルの受信チャネル：

```kotlin
interface ReceiveChannel<T> {
    suspend fun receive(): T
    suspend operator fun iterator(): ReceiveIterator<T>
}
```

The `Channel<T>` class implements both interfaces.
The `send` suspends when the channel buffer is full, while `receive` suspends when the buffer is empty.
It allows us to copy Go-style code into Kotlin almost verbatim.
The `fibonacci` function that sends `n` fibonacci numbers in to a channel from
[the 4th concurrency example of a tour of Go](https://tour.golang.org/concurrency/4)  would look 
like this in Kotlin:
Channel<T>クラスは、両方のインターフェイスを実装しています。send一時停止しながら、チャネル・バッファは、いっぱいになったときにreceiveバッファが空のときに一時停止します。Go-styleコードをほぼそのままKotlinにコピーすることができます。fibonacci送信機能nからチャネルへにフィボナッチ数を 囲碁のツアーの第四同時実行例 Kotlinに次のようになります。

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

We can also define Go-style `go {...}` block to start the new coroutine in some kind of
multi-threaded pool that dispatches an arbitrary number of light-weight coroutines onto a fixed number of 
actual heavy-weight threads.
The example implementation [here](examples/channel/go.kt) is trivially written on top of
Java's common [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html).
我々はまた、ゴースタイルの定義することができgo {...}、実際の重いスレッドの固定数に軽量コルーチンの任意の数をディスパッチマルチスレッドプールのいくつかの種類に新しいコルーチンを開始するためにブロックを。ここの実装例は、Javaの共通の上に書かれていForkJoinPoolます。

Using this `go` coroutine builder, the main function from the corresponding Go code would look like this,
where `mainBlocking` is shortcut helper function for `runBlocking` with the same pool as `go{}` uses:
このgoコルーチン構築ツールを使用すると、対応するGoコードのmain関数は次のようになります。ここでmainBlockingはrunBlocking、go{}使用と同じプールでのショートカットヘルパー関数があります。

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val c = Channel<Int>(2)
    go { fibonacci(10, c) }
    for (i in c) {
        println(i)
    }
}
```

> You can checkout working code [here](examples/channel/channel-example-4.kt)
ここで作業コードをチェックアウトすることができます

You can freely play with the buffer size of the channel. 
For simplicity, only buffered channels are implemented in the example (with a minimal buffer size of 1), 
because unbuffered channels are conceptually similar to [asynchronous sequences](#asynchronous-sequences)
that were covered before.
チャンネルのバッファーサイズで自由に演奏できます。単純化のために、バッファされていないチャネルは概念的 に前に説明した非同期シーケンスと類似しているため、この例ではバッファされたチャネルのみが実装されています（最小バッファサイズ1）。

Go-style `select` control block that suspends until one of the actions becomes available on 
one of the channels can be implemented as a Kotlin DSL, so that 
[the 5th concurrency example of a tour of Go](https://tour.golang.org/concurrency/5)  would look 
like this in Kotlin:
selectアクションの1つがチャネルの1つで利用可能になるまで保留するGoスタイルの制御ブロックは、Kotlin DSLとして実装することができます。その ため、Goのツアーの5番目の並行処理の例は、 Kotlinでは次のようになります。
 
```kotlin
suspend fun fibonacci(c: SendChannel<Int>, quit: ReceiveChannel<Int>) {
    var x = 0
    var y = 1
    whileSelect {
        c.onSend(x) {
            val next = x + y
            x = y
            y = next
            true // continue while loop// whileループを継続する
        }
        quit.onReceive {
            println("quit")
            false // break while loop//whileループを抜ける
        }
    }
}
```

> You can checkout working code [here](examples/channel/channel-example-5.kt)
ここで作業コードをチェックアウトすることができます
  
Example has an implementation of both `select {...}`, that returns the result of one of its cases like a Kotlin 
[`when` expression](https://kotlinlang.org/docs/reference/control-flow.html#when-expression), 
and a convenience `whileSelect { ... }` that is the same as `while(select<Boolean> { ... })` with fewer braces.
例には、select {...}Kotlin when式のようなケースの結果を返す 両方の実装と、より少ない中カッコwhileSelect { ... }と同じような利便性がありwhile(select<Boolean> { ... })ます。
  
The default selection case from [the 6th concurrency example of a tour of Go](https://tour.golang.org/concurrency/6) 
just adds one more case into the `select {...}` DSL:
Goのツアーの6番目の並行処理の例のデフォルトの選択例では、 もう1つのケースがselect {...}DSLに追加されます。

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val tick = Time.tick(100)
    val boom = Time.after(500)
    whileSelect {
        tick.onReceive {
            println("tick.")
            true // continue loop//ループを継続する
        }
        boom.onReceive {
            println("BOOM!")
            false // break loop
        }
        onDefault {
            println("    .")
            delay(50)
            true // continue loop
        }
    }
}
```

> You can checkout working code [here](examples/channel/channel-example-6.kt)
ここで作業コードをチェックアウトすることができます

The `Time.tick` and `Time.after` are trivially implemented 
[here](examples/channel/time.kt) with non-blocking `delay` function.
Time.tickそしてTime.after自明実装されている ここでは非ブロッキングでdelay機能。
  
Other examples can be found [here](examples/channel/) together with the links to 
the corresponding Go code in comments.
その他の例は、コメント内の対応するGoコードへのリンクと共にここにあります。

Note, that this sample implementation of channels is based on a single
lock to manage its internal wait lists. It makes it easier to understand and reason about. 
However, it never runs user code under this lock and thus it is fully concurrent. 
This lock only somewhat limits its scalability to a very large number of concurrent threads.
このチャネルの実装例は、内部待機リストを管理するための単一のロックに基づいていることに注意してください。これは、理解しやすくするための理由を簡単にします。ただし、このロックの下でユーザーコードを実行することはないため、完全に同時です。このロックは、スケーラビリティを非常に多数の同時スレッドに制限します。

> The actual implementation of channels and `select` in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  is based on lock-free disjoint-access-parallel data structures.
チャンネルの実際の実装selectでkotlinx.coroutinesは ロックフリー互いに素アクセス並列データ構造に基づいています。

This channel implementation is independent 
of the interceptor in the coroutine context. It can be used in UI applications
under an event-thread interceptor as shown in the
corresponding [continuation interceptor](#continuation-interceptor) section, or with any other one, or without
an interceptor at all (in the later case, the execution thread is determined solely by the code
of the other suspending functions used in a coroutine).
The channel implementation just provides thread-safe non-blocking suspending functions.
このチャネルの実装は、コルーチンのコンテキストのインターセプタとは独立しています。これは、対応する継続インターセプタセクションに示されているように、イベントスレッドインターセプタの下でUIアプリケーションで使用することも、インターセプタをまったく使用しないこともできます（後で実行スレッドは、コルーチンに使用される他の中断機能）。チャネル実装は、スレッドセーフなノンブロッキング中断機能を提供するだけです。
  
### Mutexes

Writing scalable asynchronous applications is a discipline that one follows, making sure that ones code 
never blocks, but suspends (using suspending functions), without actually blocking a thread.
The Java concurrency primitives like 
[`ReentrantLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)
are thread-blocking and they should not be used in a truly non-blocking code. To control access to shared
resources one can define `Mutex` class that suspends an execution of coroutine instead of blocking it.
The header of the corresponding class would like this:
スケーラブルな非同期アプリケーションを書くことは、スレッドが実際にブロックされることなく、コードがブロックされずに中断される（中断機能を使用して）ことを確実にする規律です。Java並行処理のプリミティブ ReentrantLock はスレッドブロッキングであり、真に非ブロッキングコードでは使用しないでください。共有リソースへのアクセスを制御Mutexするために、コルーチンをブロックする代わりに、コルーチンの実行を中断するクラスを定義することができます。対応するクラスのヘッダは次のようになります：

```kotlin
class Mutex {
    suspend fun lock()
    fun unlock()
}
```

> You can get full implementation [here](examples/mutex/mutex.kt).
  The actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  has a few additional functions.
ここで完全に実装することができます。で実際の実装kotlinx.coroutinesは 、いくつかの追加機能を備えています。

Using this implementation of non-blocking mutex
[the 9th concurrency example of a tour of Go](https://tour.golang.org/concurrency/9)
can be translated into Kotlin using Kotlin's
[`try-finally`](https://kotlinlang.org/docs/reference/exceptions.html)
that serves the same purpose as Go's `defer`:
この非ブロッキングミューテックス の実装を使用すると、Goのツアーの9番目の並行処理の例は、try-finally Goのものと同じ目的を果たす Kotlinのものを使用してKotlinに翻訳でき deferます。

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

> You can checkout working code [here](examples/channel/channel-example-9.kt)
ここで作業コードをチェックアウトすることができます

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
 

