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

- フューチャーなどのリッチなライブラリの特定の実装に依存しない。
- "async/await"ユースケースと "ジェネレーターブロック"を等しくカバーする。
- 異なる既存の非同期APIのラッパーとしてKotlinコルーチンを利用できるようにする（Java NIO、フューチャーのさまざまな実装など）。

## 目次

* [ユースケース](#ユースケース)
  * [非同期計算](#非同期計算)
  * [フューチャー](#フューチャー)
  * [ジェネレーター](#ジェネレーター)
  * [非同期UI](#非同期UI)
  * [その他のユースケース](#その他のユースケース)
* [コルーチンの概要](#コルーチンの概要)
  * [コルーチンの実験的状態](#コルーチンの実験的状態)
  * [用語](#用語)
  * [継続インターフェイス](#継続インターフェイス)
  * [サスペンド関数](#サスペンド関数)
  * [コルーチンビルダー](#コルーチンビルダー)
  * [コルーチンコンテキスト](#コルーチンコンテキスト)
  * [継続インターセプター](#継続インターセプター)
  * [制限付きサスペンド](#制限付きサスペンド)
* [その他の例](#その他の例)
  * [コールバックのラッピング](#コールバックのラッピング)
  * [フューチャーの作成](#フューチャーの作成)
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
  * [コルーチンの基礎](#コルーチンの基礎)
* [Revision history](#revision-history)
  * [Changes in revision 3.2](#changes-in-revision-32)
  * [Changes in revision 3.1](#changes-in-revision-31)
  * [Changes in revision 3](#changes-in-revision-3)
  * [Changes in revision 2](#changes-in-revision-2)

## ユースケース

コルーチンは _中断可能な計算_ のインスタンスとして考えることができます。ある時点で中断し、後で別のスレッドで実行を再開することができます。
コルーチンはお互いを呼び出す（そしてデータを前後に渡す）ことで、協力的なマルチタスキングのための機構を形成することができます。

### 非同期計算

コルーチンの動機付けの最初のクラスは、非同期計算です（C#やその他の言語ではasync/awaitで処理されます）。
そのような計算がコールバックでどのように行われるかを見てみましょう。インスピレーションとして、非同期I/O（以下のAPIは簡略化されています）を例にとりましょう。

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

ここではコールバック内にコールバックがあり、多くの定型文（例えば、コールバックに `buf` パラメータを明示的に渡す必要はありません。単にクロージャの一部として見るだけです）から私たちを救う一方、インデントレベルは毎回増加しています。1つ以上のネストレベルで発生する可能性のある問題を簡単に予測することができます（JavaScriptでこれでどれくらいの人が苦しんでいるかを知るために「コールバック地獄」についてググってください）。

これと同じ計算はコルーチンとして直接的に表現できます（I/O APIをコルーチンの要件に適合させたライブラリがあれば）。

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

ここでの `aRead()`と `aWrite()`は特別な _サスペンド関数_ です。実行を _中断_ し、呼び出しが完了すると _再開_ します。
`aRead()`の後のすべてのコードがラムダでラップされ、コールバックとして`aRead()`に渡され、`aWrite()`に対して同じことが行われたと想像するだけで、 私たちはこのコードが上記と同じで、より読み易いことが分かります。

これはコルーチンを非常に一般的な方法でサポートするという私たちの明白な目標です。この例では、 `launch{}`、 `.aRead()`、 `.aWrite()`は単にコルーチンを扱う **ライブラリ関数です** 。
`aRead`/`aWrite`は暗黙的に _継続_ （単なる一般的なコールバック）を受け取る特殊な _サスペンド関数_ ですが、`launch` （ _コルーチンビルダー_ ）はコルーチンをコンテキストの中で（この例では `CommonPool`コンテキストが使用されています）でビルドして起動します。

> `launch{}`のライブラリコードは[コルーチンビルダー](#コルーチンビルダー)セクションに、 `.aRead()`のライブラリコードは[コールバックのラッピング](#コールバックのラッピング)セクションに示されています。

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

### フューチャー

非同期計算を表現するフューチャー（そして近い関係のプロミス）を使った別のスタイルがあります。
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
    // `applyOverlay(...)` を実行する
    applyOverlay(original.await(), overlay.await())
}
```

> `future{}`のライブラリコードは[フューチャーの作成](#フューチャーの作成)セクションに、`.await()`のライブラリコードは[サスペンド関数](#サスペンド関数)に示されています。

繰り返しますが、字下げが少なく自然な構成ロジック（ここでは示しませんが例外処理も）で、フューチャーをサポートするための特別なキーワード（C#, JSなどの言語での`async`や`await`のような）もありません。
`future{}`や`.await()`はただのライブラリ関数です。

### ジェネレーター

コルーチンの別の典型的な使用例は、遅延計算シーケンス（C#、Python、その他多くの言語で`yield`によって処理される）です。
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

このコードは、潜在的に無限にある[フィボナッチ数](https://ja.wikipedia.org/wiki/%E3%83%95%E3%82%A3%E3%83%9C%E3%83%8A%E3%83%83%E3%83%81%E6%95%B0)の遅延 `Sequence` を作成します（[Haskelの無限リスト](http://www.techrepublic.com/article/infinite-list-tricks-in-haskell/)のような）。
たとえば、`take()`を使ってその一部を要求することができます。

```kotlin
println(fibonacci.take(10).joinToString())
```

> これは`1, 1, 2, 3, 5, 8, 13, 21, 34, 55`をプリントします。
  [ここ](examples/sequence/fibonacci.kt)でこのコードを試すことができます。

ジェネレーターの長所は、`while`（上記の例から）、`if`、`try`/`catch`/`finally`などの任意の制御フローをサポートしているところです。

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
[制限付きサスペンド](#制限付きサスペンド)のセクションに示されています。

この方法では、ライブラリ関数（`buildSequence{}`と `yield()`も同様）で `yieldAll(sequence)` と表現することもできます。これは、遅延配列の結合を簡単にし、効率的な実装を可能にします。

### 非同期UI

一般的なUIアプリケーションには、すべてのUI操作が発生する単一のイベントディスパッチスレッドがあります。
他のスレッドからのUI状態の変更は通常許可されません。すべてのUIライブラリは、実行をUIスレッドに戻すためのプリミティブを提供します。たとえばSwingには [`SwingUtilities.invokeLater`](https://docs.oracle.com/javase/jp/8/docs/api/javax/swing/SwingUtilities.html#invokeLater-java.lang.Runnable-)、JavaFXには [`Platform.runLater`](https://docs.oracle.com/javase/jp/8/javafx/api/javafx/application/Platform.html#runLater-java.lang.Runnable-)、Androidには [`Activity.runOnUiThread`](https://developer.android.com/reference/android/app/Activity.html#runOnUiThread(java.lang.Runnable))など。
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

これは、[非同期計算](#非同期計算)ユースケースのコールバック地獄に似ていますが、これもコルーチンによってエレガントに解決されています。

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

> `Swing`コンテキストのライブラリコードは[継続インターセプター](#継続インターセプター)セクションに示されています。

すべての例外処理は、自然な言語構造を使用して実行されます。

### その他のユースケース

コルーチンは、次のような多くのユースケースをカバーできます。

* チャネルベースの同時実行性（goroutinesとチャネルとも呼ばれます）
* アクターベースの同時実行性
* ユーザとの対話を必要とするバックグラウンドプロセス。たとえば、モーダルダイアログの表示
* 通信プロトコル: 各アクターを状態マシンではなくシーケンスとして実装する
* Webアプリケーションのワークフロー: ユーザーの登録、電子メールのバリデーション、ログイン
（サスペンドされたコルーチンを直列化してDBに格納することができます）

## コルーチンの概要

このセクションでは、コルーチンとそのセマンティクスを制御する標準ライブラリを記述するための言語メカニズムの概要を説明します。

### コルーチンの実験的状態

コルーチンはKotlin 1.1で実験的です。なぜなら、我々は設計が変わることを想定しているからです。Kotlinコンパイラは、コルーチン関連の機能の使用に関する警告を出します。警告を取り除く `-Xcoroutines=enable` オプトインスイッチがあります 。

kotlin-stdlibのコルーチンに関連するすべてのAPIは、`kotlin.coroutines.experimental` パッケージに入っています。最終的な設計では、`kotlin.coroutines` パッケージで公開されますが、実験的なパッケージはしばらくの間そのままなので、古いバイナリは互換性があり作業を続けられます。

パブリックAPIでコルーチンを使用するすべてのライブラリで同じことを行う必要があります。したがって、ここでライブラリを作成しておき、将来のバージョンのユーザーを気にかける場合は、パッケージに `org.my.library.experimental` のような名前を付ける必要があります 。
そして、コルーチンの最終的な設計が来たら、メインAPIから `experimental` サフィックスを削除しますが、バイナリ互換性が必要なユーザーのために古いパッケージを残しておきます。

> 詳細は、[この](https://discuss.kotlinlang.org/t/experimental-status-of-coroutines-in-1-1-and-related-compatibility-concerns/2236) フォーラムの投稿で見つけることができます

### 用語

* _コルーチン_ - _中断可能な計算_ の _インスタンス_ です。
  概念的にはスレッドに似ていて、実行するコードブロックを持ち、同様のライフサイクルを持ち、_作成_ され _起動_ されますが、特定のスレッドに束縛されていません。あるスレッドで実行を _中断_ し、別のスレッドで _再開_ することがあります。
  さらに、フューチャーやプロミスのように、何らかの結果や例外で _完了_ します。
 
* _サスペンド関数_ - `suspend` 修飾子でマークされた関数。
  他のサスペンド関数を呼び出すことによって、現在の実行スレッドをブロックせずにコードの実行を _中断_ することができます。
  サスペンド関数は、通常のコードから呼び出すことはできませんが、他のサスペンド関数やサスペンドラムダからのみ呼び出すことができます（下記参照）。
  例えば、 [ユースケース](#ユースケース)で示されている `.await()` や `yield()` は、ライブラリに定義されているサスペンド関数です。
  標準ライブラリは、他のすべてのサスペンド関数を定義するために使用されるプリミティブなサスペンド関数を提供します。

* _サスペンドラムダ_ - コルーチンで実行できるコードブロック。
  通常の[ラムダ式](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/lambdas.html)と全く同じように見えますが、 その関数型は `suspend` 修飾子でマークされています。
  通常のラムダ式と同様に、匿名のローカル関数用の短い構文形式です。サスペンドラムダは、匿名のサスペンド関数の短い構文形式です。
  サスペンド関数を呼び出すことによって、現在の実行スレッドをブロックせずにコードの実行を _中断_ することができます。
  例えば、 [ユースケース](#ユースケース)で示されている `launch`、` future`、`buildSequence` 関数に続く中括弧で囲まれたコードブロックは、サスペンドラムダです。

> 注：サスペンドラムダは、このラムダの[非ローカル](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/returns.html) `return` 文が許可されているコードのすべての場所でサスペンド関数を呼び出せます。
  つまり、 [`apply{}` ブロック](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html) のようなインラインラムダの中で関数呼び出しを中断することは許されますが、 `noinline` や `crossinline` の内部ラムダ式では許可されません。
  _中断_ は特別な種類の非ローカル制御転送として扱われます。

* _サスペンド関数型_ - サスペンド関数とサスペンドラムダの関数型です。
  通常の[関数型](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/lambdas.html#関数型)に似ていますが、`suspend` 修飾子が付いています。
  たとえば、`suspend() -> Int` は `Int` を返す引数のないサスペンド関数の型です。
  `suspend fun foo(): Int` のように宣言されたサスペンド関数は、この関数型に準拠しています。

* _コルーチンビルダー_ - 引数としていくつかの _サスペンドラムダ_ を取り、コルーチンを作成し、オプションでその結果へのアクセスを与える関数。
  たとえば、 ユースケースで示した `launch{}`、`future{}`、`buildSequence{}` は、ライブラリに定義されているコルーチンのビルダーです。
  標準ライブラリは、他のすべてのコルーチンビルダーを定義するためのプリミティブなコルーチンビルダーを提供します。

> 注：一部の言語では、実行と結果の表現方法を定義するコルーチンを作成して開始する特定の方法をハードコーディングでサポートしています。
たとえば、`generate` _キーワード_ はある種の反復可能オブジェクトを返すコルーチンを定義し、` async` _キーワード_ はある種のpromiseやタスクを返すコルーチンを定義します。
Kotlinには、コルーチンを定義して開始するためのキーワードや修飾子はありません。
コルーチンのビルダーは、単にライブラリ内で定義された関数です。
コルーチンの定義が別の言語のメソッド本体の形を取っている場合、Kotlinでは典型的に、式本体を持つ通常のメソッドであり、最後の引数がサスペンドラムダであるライブラリ定義のコルーチンビルダーの呼び出しから成ります。
 
```kotlin
fun asyncTask() = async { ... }
```

* 中断ポイント_ - コルーチンの実行が _中断される可能性がある_ ポイントです。
構文的には、中断ポイントはサスペンド関数の呼び出しですが、サスペンド関数が標準ライブラリプリミティブを呼び出して実行を中断すると、実際の中断が発生します。

* _継続_ - 中断ポイントで中断したコルーチンの状態です。中断ポイントのあとの残りの実行を概念的に表しています。例えば、

```kotlin
buildSequence {
    for (i in 1..10) yield(i * i)
    println("over")
}  
```  

ここでは、コルーチンがサスペンド関数 `yield()` の呼び出しで中断されるたびに _残りの実行_ は継続として表されるので、10個の継続があります。
最初にループを実行して `i = 2` で中断し、2番目にループを実行して `i = 3` で中断し、最後のコマンドで "over" を出力してコルーチンを完成させます。
_作成_ されてまだ _開始_ されていないコルーチンは、その実行全体からなる `Continuation<Unit>' 型の初期の継続によって表されます。

前述したように、コルーチンの駆動要件の1つは柔軟性です。既存の多くの非同期APIやその他のユースケースをサポートし、コンパイラーにハードコードされた部分を最小限に抑えたいと考えています。その結果コンパイラは、サスペンド関数、サスペンドラムダ、および対応するサスペンド関数型のサポートのみを行います。標準ライブラリにはいくつかのプリミティブがあり、残りはアプリケーションライブラリに残されています。

### 継続インターフェイス

一般的なコールバックを表す標準ライブラリインターフェイス `Continuation` の定義を次に示します。

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

`.await()` のような典型的な _サスペンド関数_ の実装は次のようになります。
  
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

> [ここ](examples/future/await.kt)でこのコードを入手できます。注：この簡単な実装は、フューチャーが完了しなければ、コルーチンを永久に中断します。実際の[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実装は取り消しをサポートするのでやや複雑です。

`suspend` 修飾子は、コルーチンの実行を中断することができる関数であることを表しています。
この特定の関数は、`CompletableFuture<T>` 型の[拡張関数](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/extensions.html)として定義され、実際の実行順序に対応して左から右の順に自然に読みます。

```kotlin
asyncOperation(...).await()
```
 
修飾子 `suspend` は、トップレベル関数、拡張関数、メンバー関数、または演算子関数のいずれかの関数で使用できます。

> 現在のリリースではローカル関数、プロパティのゲッター/セッターおよびコンストラクタに `suspend` 修飾子を付けることはできません。これらの規制は将来廃止される予定です。
 
サスペンド関数は通常の関数を呼び出すことができますが、実際に実行を中断するには他のサスペンド関数を呼び出す必要があります。
具体的には、この `await` 実装は次のようにトップレベルのサスペンド関数として標準ライブラリに定義されたサスペンド関数 ` suspendCoroutine` を呼び出します。

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

コルーチンの内部で `suspendCoroutine` が呼び出されると（これはサスペンド関数であるため、コルーチン内でのみ呼び出すことができます）、コルーチンの実行を _中断_ し、_継続_ インスタンスにその状態をキャプチャして、この継続を指定された`block`に引数として渡します。
コルーチンの実行を再開するために、ブロックは他のスレッドでこのスレッドの `continuation.resume()` または `continuation.resumeWithException()` を呼び出すことができます。
同じ継続を2回以上再開することは許されず、`IllegalStateException` を生成します。

> 注：これは、Kotlinのコルーチンと、SchemeやHaskelの継続モナドのような関数言語のファーストクラスの限定継続との主要な違いです。
制限された一度だけの再開のみのサポートを選択することは純粋に実用的で、意図された[ユースケース](#ユースケース)がファーストクラスの継続を必要としないため、それらの限定バージョンをより効率的に実装することができます。
しかし、ファーストクラスの継続は、継続にキャプチャされたコルーチンの状態を複製することによって別のライブラリとして実装することができ、その複製を再び再開することができます。
このメカニズムは、将来、標準ライブラリによって効率的に提供される可能性があります。

`continuation.resume()` に渡された値は `suspendCoroutine()` の**戻り値**になり、コルーチンが実行を _再開_ すると `.await()` の戻り値になります。

### コルーチンビルダー

サスペンド関数は通常の関数から呼び出すことができないので、標準ライブラリは通常の非サスペンドスコープからコルーチンの実行を開始する関数を提供します。シンプルな `launch{}` _コルーチンビルダー_ の実装を以下に示します。

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

この実装は、このコルーチンを表す単純なクラス `StandaloneCoroutine` を定義し、その完了を取得するための `Continuation` インターフェイスを実装します。
コルーチンの完了はその _完了の継続_ を呼び出します。
`resume` または `resumeWithException` 関数は、コルーチンの結果または例外での _完了_ に対応して呼び出されます。
`launch`はコルーチンを「撃ち放し」にするため、戻り値の型は `Unit` 型でサスペンド関数が定義され、実際には `resume` 関数でこの結果を無視します。
コルーチンの実行が例外で完了すると、現在のスレッドのキャッチされていない例外ハンドラがそれを報告するために使用されます。

> 注：この単純な実装は `Unit` を返し、コルーチンの状態へのアクセスをまったく提供しません。
[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)の実際の実装はより複雑です。なぜなら、コルーチンを表す `Job`インターフェイスのインスタンスを返し、取り消すことができるからです。

コンテキストは、[コルーチンコンテキスト](#コルーチンコンテキスト)のセクションで詳しく説明されています。
ここでは、有用なコンテキスト要素を定義する可能性がある他のライブラリとのより良い _合成_ のために、ライブラリ定義のコルーチンビルダーに `context` パラメータを含めることは良いスタイルだと言えば十分です。

`startCoroutine` は、スタンダードライブラリ内で、サスペンド関数型の拡張として定義されています。
そのシグネチャは次のとおりです。

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
```

`startCoroutine`はコルーチンを作成し、現在のスレッドですぐに最初の _中断ポイント_ まで実行を開始し（以下の注釈を参照してください）、リターンします。
中断ポイントはコルーチン本体での[サスペンド関数](#サスペンド関数)の呼び出しで、コルーチンの実行がいつどのように再開するかは、対応するサスペンド関数のコードによって決まります。

> 注：後で説明する[継続インターセプター](#継続インターセプター)（コンテキストから）は、最初の継続を _含む_ コルーチンの実行を別のスレッドにディスパッチできます。

### コルーチンコンテキスト

コルーチンコンテキストは、コルーチンに添付できる永続的なユーザ定義オブジェクトのセットです。これは、コルーチンのスレッドポリシー、ロギング、コルーチンの実行のセキュリティとトランザクションのアスペクト、コルーチンの識別子と名前などを担当するオブジェクトを含むことができます。コルーチンとそのコンテキストの単純なメンタルモデルがあります。コルーチンを軽量スレッドと考えてください。この場合、コルーチンのコンテキストは、スレッドローカル変数の集合のようなものです。
相違点は、スレッドローカル変数は変更可能でコルーチンのコンテキストは不変であることですが、コンテキストの中で何かを変更する必要があるときに新しいコルーチンを起動するのは簡単で軽量であるため、コルーチンの重大な制限ではありません。

標準ライブラリには、コンテキスト要素の具体的な実装は含まれていませんが、インターフェイスと抽象クラスがあり、これらのすべてのアスペクトをライブラリで _構成可能_ な方法で定義することができるため、異なるライブラリからのアスペクトを同じ文脈の要素として平和的に共存させることができます。

概念的には、コルーチンのコンテキストは、各要素に一意のキーを持つ要素のインデックス付きセットです。これは、セットとマップのミックスです。その要素はマップのようなキーを持っていますが、キーは要素に直接関連付けられています。標準ライブラリは、`CoroutineContext` のための最小限のインターフェイスを定義します。

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

* `get` 演算子は、指定されたキーの要素への型セーフなアクセスを提供します。これは、[Kotlin演算子のオーバーロード](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/operator-overloading.html)で説明した `[..]` 表記法で使用できます。
* `fold` 関数は 標準ライブラリの [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) 拡張のように機能し、コンテキスト内のすべての要素を反復する手段を提供します。
* 演算子 `plus` は 標準ライブラリの [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html) 拡張のように機能し、2つのコンテキストの組みの左側の要素を同じキーの右側の要素で置き換えて返します。
* 関数 `minusKey` は、指定されたキーを含まないコンテキストを返します。

コルーチンコンテキストの `Element` はコンテキスト自体です。 この要素のみを持つシングルトンコンテキストです。
これにより、コルーチンコンテキスト要素のライブラリ定義を取得し、それらを `+` で結合することによって、複合コンテキストを作成することができます。
たとえば、あるライブラリがユーザー認証情報を持つ `auth` 要素を定義し、いくつかの他のライブラリがいくつかの実行コンテキスト情報を持つ `CommonPool` オブジェクトを定義している場合、`launch(auth + CommonPool) {...}` を呼び出すことでコンテキストを組み合わせた `launch{}` [コルーチンビルダー](#コルーチンビルダー)を使うことができます。

> 注：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)は、コルーチンの実行をバックグラウンドスレッドの共有プールにディスパッチする `CommonPool` オブジェクトを含むいくつかのコンテキスト要素を提供します。

すべてのライブラリ定義のコンテキスト要素は、標準ライブラリによって提供される `AbstractCoroutineContextElement` クラスを継承するものとします。
ライブラリ定義のコンテキスト要素には、次のスタイルが推奨されます。
次の例は、現在のユーザー名を格納する仮の認可コンテキスト要素を示しています。

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

対応する要素クラスのコンパニオンオブジェクトとしてのコンテキスト `Key` の定義は、コンテキストの対応する要素への流暢なアクセスを可能にします。
現在のユーザーの名前を確認する必要があるサスペンド関数の仮説的実装です。

```kotlin
suspend fun secureAwait(): Unit = suspendCoroutine { cont ->
    val currentUser = cont.context[AuthUser]?.name
    // ユーザー固有の処理を行う 
}
```

### 継続インターセプター

[非同期UI](#非同期UI)のユースケースを要約しましょう。非同期UIアプリケーションでは、さまざまなサスペンド関数が任意のスレッドでコルーチンの実行を再開しても、コルーチン本体自体が常にUIスレッドで実行されるようにする必要があります。これは、 _継続インターセプター_ を使用して達成されます。まず、コルーチンのライフサイクルを完全に理解する必要があります。[`launch{}`](#コルーチンビルダー)コルーチンビルダーを使用したコードスニペットを考えてみましょう 。

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
中断ポイントでは _中断_ し、しばらくして対応するサスペンド関数で定義されたように `block1` の実行に _復帰_ し、再び中断して `block2` の実行に復帰した後、 _完了_ します。

継続インターセプターには `initialCode`、` block1`、および `block2` の実行に対応する継続を、それらの後続の中断ポイントへの再開からインターセプトしてラップするオプションがあります。
コルーチンの初期コードは、 _初期継続_ の再開として扱われます。
標準ライブラリは、次のインターフェイスを提供します。
 
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
 
コルーチンフレームワークは、継続のそれぞれの実際のインスタンスの結果としてのファサードをキャッシュします。詳細については、 [実装の詳細](#実装の詳細)のセクションを参照してください。

実行をSwing UIイベントディスパッチスレッドにディスパッチする `Swing` インターセプターの具体的なコード例を見てみましょう。
現在のスレッドをチェックし、継続がSwingイベントディスパッチスレッドでのみ再開することを確実にする `SwingContinuation` ラッパークラスの定義から始めます。
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

次に、対応するコンテキスト要素として機能する `Swing` オブジェクトを定義し、` ContinuationInterceptor` インターフェイスを実装します。
  
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

これは、コルーチンを _作成_ する `createCoroutine` という標準ライブラリの異なるプリミティブを使用しますが、起動 _しません_ 。
代わりに、`Continuation<Unit>` への参照として _最初の継続_ を返します。
もう1つの違いは、このビルダーの _サスペンドラムダ_  `block` は `SequenceBuilder<T>` レシーバーを持つ[_拡張ラムダ_](http://dogwood008.github.io/kotlin-web-site-ja/docs/reference/lambdas.html#レシーバ付き関数リテラル)です。
`SequenceBuilder` インターフェイスは、ジェネレーターブロックの _スコープ_ を提供し、ライブラリとして次のように定義されています。

```kotlin
interface SequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

複数のオブジェクトの作成を避けるため、`buildSequence{}`実装は `SequenceBuilder<T>` を実装する `SequenceCoroutine<T>` クラスを定義し、`Continuation<Unit>` も実装していますので、`createCoroutine` の `receiver` パラメータ としても `completion` 継続パラメータとしても機能します。
`SequenceCoroutine<T>` の単純な実装を以下に示します。

```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceBuilder<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // AbstractIteratorの実装
    override fun computeNext() { nextStep.resume(Unit) }

    // 完了継続の実装
    override val context: CoroutineContext get() = EmptyCoroutineContext
    override fun resume(value: Unit) { done() }
    override fun resumeWithException(exception: Throwable) { throw exception }

    // Generatorの実装
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```
 
> [ここ](examples/sequence/buildSequence.kt)でこのコードを入手できます

`yield`の実装は `suspendCoroutine` [サスペンド関数](#サスペンド関数)を使ってコルーチンを中断し、その継続を捕捉します。
`computeNext` が呼び出されると、継続は `nextStep` として保存され、再開されます。
 
しかし、上に示した `buildSequence{}` と `yield()` は、任意のサスペンド関数がそのスコープ内の継続を捕捉する準備ができていません。それらは _同期_ して動作します。
継続をどのようにキャプチャし、どこに格納し、いつ再開するかを絶対的に制御する必要があります。
_制限付きサスペンドスコープ_ を形成します。
中断を制限する機能は、スコープクラスまたはインターフェイスに配置された `@RestrictsSuspension` アノテーションによって提供されます。上記の例では、このスコープインターフェイスは `SequenceBuilder` です。

```kotlin
@RestrictsSuspension
interface SequenceBuilder<in T> {
    suspend fun yield(value: T)
}
```

このアノテーションは、`SequenceBuilder{}` または同様の同期コルーチンビルダーのスコープで使用できるサスペンド関数に一定の制限を適用します。
_制限付きサスペンドスコープ_ のクラスまたはインターフェイス（`@RestrictsSuspension` でマークされている）をレシーバーとして持つサスペンドラムダまたはサスペンド関数の拡張は、 _制限付きサスペンド関数_ と呼ばれます。
制限付きサスペンド関数は、同じ制限付きサスペンドスコープのインスタンスの中でのみメンバ関数または拡張関数を呼び出すことができます。
具体的には、スコープ内のラムダの `SequenceBuilder` 拡張が `suspendContinuation` やその他の一般的なサスペンド関数を呼び出せないことを意味します。
`generate` コルーチンの実行を中断するには、最終的に `SequenceBuilder.yield` を呼び出さなければなりません。
`yield` 自体の実装は `Generator` 実装のメンバー関数であり、何の制約もありません（ _拡張_ サスペンドラムダと関数のみ制限があります）。

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
これは、Java NIO [`AsynchronousFileChannel`](http://docs.oracle.com/javase/jp/8/docs/api/java/nio/channels/AsynchronousFileChannel.html) と [`CompletionHandler`](http://docs.oracle.com/javase/jp/8/docs/api/java/nio/channels/CompletionHandler.html) コールバックインターフェイスのためのサスペンド拡張関数として次のコードで実装できます。

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

### フューチャーの作成

[フューチャー](#フューチャー)ユースケースの `future{}` ビルダーは、[コルーチンビルダー](#コルーチンビルダー)のセクションで説明したように、フューチャーまたはプロミスのプリミティブに対して `launch{}` ビルダーと同様に定義することができます。

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

このコルーチンの完了は、このコルーチンの結果を記録するため、フューチャーの対応する `complete` メソッドを呼び出します。

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

以下の、内部にアクティブな2つの非同期タスクを持っているにもかかわらず単一のスレッドで動作する例の中で、[フューチャーの作成](#フューチャーの作成)セクションで定義された `future{}` コルーチンビルダーとともに`newSingleThreadContext()` 関数を使用します。

```kotlin
fun main(args: Array<String>) {
    log("Starting MyEventThread")
    val context = newSingleThreadContext("MyEventThread")
    val f = future(context) {
        log("Hello, world!")
        val f1 = future(context) {
            log("f1 is sleeping")
            delay(1000) // 1秒スリープ
            log("f1 returns 1")
            1
        }
        val f2 = future(context) {
            log("f2 is sleeping")
            delay(1000) // 1秒スリープ
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

[制限付きサスペンド](#制限付きサスペンド)セクションに示されている `buildSequence{}` コルーチンビルダーは、 _同期_ コルーチンの例です。
コルーチンのプロデューサーコードは、コンシューマーが `Iterator.next()` を呼び出すと同時に同じスレッドで同期的に呼び出されます。
`buildSequence{}` コルーチンブロックは制限されており、[コールバックのラッピング](#コールバックのラッピング)セクションに示すように、非同期ファイルIOのようなサードパーティのサスペンド関数を使って実行を中断することはできません。

_非同期_ シーケンスビルダーは、その実行を任意に中断して再開することができます。
データがまだ作成されていない場合、そのコンシューマーはそのケースを処理する準備ができていることを意味します。
これはサスペンド関数の自然な使用例です。通常の [`Iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterator/)インターフェイスに似た `SuspendingIterator` インターフェイスを定義しましょう。ただし、`next()` と `hasNext()` 関数は中断できます。
 
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

Goスタイルのタイプセーフなチャンネルは、Kotlinでライブラリとして実装できます。サスペンド関数 `send` として送信チャネルのインターフェイスを定義することができます。

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

`Channel<T>` クラスは両方のインターフェイスを実装しています。
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

このチャネルの実装例は、内部待機リストを管理するための単一のロックに基づいていることに注意してください。これは、理解しやすく、理由を簡単にします。ただし、このロックの下でユーザーコードを実行することはないため、完全に並列です。このロックは、スケーラビリティを非常に多数の並列スレッドに制限します。

> [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)でのチャネルと `select` の実際の実装は、ロックフリーのdisjoint-access-parallelデータ構造に基づいています。

このチャネルの実装は、コルーチンのコンテキストのインターセプターとは独立しています。
これは、対応する継続インターセプターのセクションに示されているように、イベントスレッドインターセプターの下でUIアプリケーションで使用することも、他のインターセプターで使用することも、インターセプターをまったく使用しないこともできます（後者の場合、実行スレッドは、コルーチンで使用される他のサスペンド関数のコードによってのみ決定されます）。
チャネル実装は、スレッドセーフなノンブロッキングサスペンド関数を提供するだけです。

### ミューテックス

スケーラブルな非同期アプリケーションを書くことは、実際にスレッドがブロックされることなく、コードがブロックされずに中断される（サスペンド関数を使用して）ことを確実にする規律です。
[`ReentrantLock`](https://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)のようなJava並行処理プリミティブはスレッドブロッキングであり、本当にノンブロッキングコードで使用すべきではありません。
共有リソースへのアクセスを制御するために、コルーチンをブロックするのではなく実行を中断する `Mutex` クラスを定義することができます。
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

## 詳細トピック

このセクションでは、リソース管理、同時実行性、およびプログラミングスタイルに関するいくつかの高度なトピックについて説明します。

### リソース管理とGC

コルーチンは、オフヒープストレージを使用せず、コルーチン内で実行されているコードがファイルやその他のリソースを開かない限り、それ自身ではネイティブリソースを消費しません。
コルーチン内で開かれたファイルは何とか閉じなければなりませんが、コルーチン自体を閉じる必要はありません。
コルーチンが中断されている場合、コルーチン全体の状態はその継続を参照することで利用可能です。
中断されたコルーチンの継続への参照を失った場合、最終的にはガベージコレクターによって収集されます。

クローザブルリソースを開くコルーチンは特別な注意が必要です。[制限付きサスペンド](#制限付きサスペンド)セクションの `buildSequence{}` ビルダーを使用してファイルから行のシーケンスを生成する、次のコルーチンを考えてみましょう。

```kotlin
fun sequenceOfLines(fileName: String) = buildSequence<String> {
    BufferedReader(FileReader(fileName)).use {
        while (true) {
            yield(it.readLine() ?: break)
        }
    }
}
```

この関数は `Sequence<String>` を返し、この関数を使って自然な方法でファイルからすべての行を出力することができます。
 
```kotlin
sequenceOfLines("examples/sequence/sequenceOfLines.kt")
    .forEach(::println)
```

> [ここ](examples/sequence/sequenceOfLines.kt)で完全なコードを入手できます

`sequenceOfLines` 関数が返すシーケンスを最後まで反復する限り、期待どおりに動作します。しかし、このようにこのファイルから最初の行を数行だけ印刷すると、

```kotlin
sequenceOfLines("examples/sequence/sequenceOfLines.kt")
        .take(3)
        .forEach(::println)
```

コルーチンは数回再開して最初の3行を生成し、 _放棄_ されます。コルーチン自体は放棄されてもOKですが、開かれたファイルはそうではありません。この [`use`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 関数は、実行を終了してファイルを閉じる機会はありません。Javaのファイルは `finalizer` でファイルを閉じるため、GCによって収集されるまでファイルは開いたままになります。小さなスライドウェアや短時間実行されるユーティリティでは大きな問題ではありませんが、数ギガバイトのヒープを持つ大規模なバックエンドシステムでは、メモリを使い切ってGCを起動するよりも速く、開いているファイルハンドルが使い果たされる可能性があります。

これは、Javaの [`Files.lines`](https://docs.oracle.com/javase/jp/8/docs/api/java/nio/file/Files.html#lines-java.nio.file.Path-)メソッドと同様のやり方で、遅延した行を生成します。
クローザブルJavaストリームを返しますが、ほとんどのストリーム操作では対応する `Stream.close` メソッドが自動的に呼び出されるわけではなく、対応するストリームを閉じなければならないことを覚えておく必要があります。
Kotlinではクローザブルシーケンスジェネレーターを定義できますが、言語上の自動メカニズムでは使用後に閉じられることを保証できない同様の問題に悩まされます。
自動化されたリソース管理のための言語メカニズムを導入することは、明示的にKotlinコルーチンの範囲外です。

しかし、通常この問題はコルーチンの非同期ユースケースには影響しません。非同期コルーチンは放棄されることはなく最終的に完了するまで実行されるため、コルーチン内のコードが適切にリソースを閉じると最終的にクローズされます。

### 並行処理とスレッド

個々のコルーチンはスレッドと同様に順次実行されます。これは、次のような種類のコードがコルーチン内で完全に安全であることを意味します。

```kotlin
launch(CommonPool) { // コルーチンを開始
    val m = mutableMapOf<String, String>()
    val v1 = someAsyncTask1().await() // awaitで中断
    m["k1"] = v1 // 再開したらマップを変更
    val v2 = someAsyncTask2().await() // awaitで中断
    m["k2"] = v2 // 再開したらマップを変更
}
```

特定のコルーチンのスコープ内で、すべての通常のシングルスレッドのミュータブル構造を使用できます。
しかし、コルーチン _間_ でミュータブルな状態を共有することは潜在的に危険です。 
ディスパッチャをインストールするコルーチンビルダーを使用して、[継続インターセプター](#継続インターセプター)セクションで示される `Swing` インターセプターのように単一のイベントディスパッチスレッドですべてのJSスタイルのコルーチンを再開すると、このイベントディスパッチスレッドから一般に変更されたすべての共有オブジェクトで安全に作業できます 
ただし、マルチスレッド環境で作業している場合や、別のスレッドで実行されているコルーチン間でミュータブルな状態を共有している場合は、スレッドセーフ（並行）データ構造を使用する必要があります。

コルーチンはもっと軽量ですがスレッドのようなものです。ほんの数個のスレッドで数百万のコルーチンを動かすことができます。実行中のコルーチンは、常に何らかのスレッドで実行されます。しかし、 _中断_ されているコルーチンはスレッドを消費せず、スレッドにバインドされません。
このコルーチンを再開するサスペンド関数は、このスレッドで `Continuation.resume` を呼び出してコルーチンが再開されるスレッドを決定し、コルーチンのインターセプターはこの決定をオーバーライドしてコルーチンの実行を別のスレッドにディスパッチできます。

## 非同期プログラミングスタイル

さまざまなスタイルの非同期プログラミングがあります。

コールバックは[非同期計算](#非同期計算)セクションで議論され、一般に、コルーチンが置き換えるように設計されている最も簡単なスタイルです。コールバックスタイルのAPIは、[ここ](#コールバックのラッピング)に示すように、対応するサスペンド関数にラップすることができます。

要約しましょう。たとえば、次のシグネチャで仮想的なブロッキング `sendEmail` 関数を使い始めるとします。

```kotlin
fun sendEmail(emailArgs: EmailArgs): EmailResult
```

これは動作中に実行スレッドを潜在的に長時間ブロックします。

ノンブロッキングにするには、たとえば、エラーファーストの [node.jsコールバック規約](https://www.tutorialspoint.com/nodejs/nodejs_callbacks_concept.htm)を使用して、ノンブロッキングバージョンをコールバックスタイルで次のシグネチャで表すことができます。

```kotlin
fun sendEmail(emailArgs: EmailArgs, callback: (Throwable?, EmailResult?) -> Unit)
```

しかし、コルーチンは、他のスタイルの非同期ノンブロッキングプログラミングを可能にします。それらの1つは、多くの一般的な言語に組み込まれているasync/awaitスタイルです。
Kotlinでは、[フューチャー](#フューチャー)のユースケースセクションの一部として示された `future{}` と `.await()` ライブラリ関数を導入することによって、このスタイルを再現することができます。

このスタイルは、コールバックをパラメータとして使用するのではなく、関数から何らかのフューチャーオブジェクトを返すように慣習によって示されています。この非同期スタイルでは、`sendEmail` のシグネチャは次のようになります。

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult>
```

スタイルの問題として、それらのパラメータがブロッキング版と変わらず、操作の非同期性を忘れて間違いやすいので、このようなメソッド名に `Async` サフィックスを追加することは良い習慣です。
関数 `sendEmailAsync` は、 _並行_ 非同期操作を開始し、潜在的に並行処理の落とし穴をもたらします。
しかし、このスタイルのプログラミングを促進する言語は、通常、必要に応じて実行をシーケンスに戻すために、ある種の `await` プリミティブを持っています。

Kotlinの _ネイティブ_ プログラミングスタイルは、サスペンド関数に基づいています。
このスタイルでは、`sendEmail` のシグネチャは自然に見えます。パラメータや戻り値の型に変更はありませんが、追加の `suspend` 修飾子が付いています。

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult
```

非同期スタイルとサスペンドスタイルは、既に見たプリミティブを使用して簡単に相互に変換できます。
例えば、`sendEmailAsync` は `sendEmail` を [`future` コルーチンビルダー](#フューチャーの作成)を使って中断することで実装できます。

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult> = future {
    sendEmail(emailArgs)
}
```

サスペンド関数 `sendEmail` は [`.await()` サスペンド関数](#サスペンド関数)を使って `sendEmailAsync` によって実装することができます。

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult = 
    sendEmailAsync(emailArgs).await()
```

したがって、ある意味では、これらの2つのスタイルは同等であり、どちらも利便性においてコールバックスタイルよりも優れています。
`sendEmailAsync` と サスペンド `sendEmail` の違いをもっと深く見てみましょう。

最初にどのように**構成**するかを比較してみましょう。サスペンド関数は、通常の関数と同様に構成することができます。

```kotlin
suspend fun largerBusinessProcess() {
    // ここにたくさんのコードがある
    sendEmail(emailArgs)
    // その後に何かが続く
}
```

対応する非同期スタイルの関数は次のように構成されます。

```kotlin
fun largerBusinessProcessAsync() = future {
   // ここにたくさんのコードがある
   sendEmailAsync(emailArgs).await()
   // その後に何かが続く 
}
```

非同期スタイル関数の構成はより冗長で _エラーが発生しやすい_ ことに注意してください。
非同期スタイルの例で `.await（）`呼び出しを省略すると、コードはコンパイルされて動作しますが、電子メールはプロセスを非同期的に、または大規模なビジネスプロセスの他の部分と _並行して_ 送信します。潜在的に共有状態を変更し、非常に再現が難しいエラーを取り込みます。
逆に、サスペンド関数は _デフォルトでは順次実行されます_ 。
サスペンド関数では、並行性が必要なときはいつでも、ある種の `future{}` や同様のコルーチンビルダー呼び出しをソースコードに明示します。

多くのライブラリを使用する大きなプロジェクトで、これらのスタイルの**スケール**をどのように比較するか。サスペンド関数は、Kotlinの軽量言語の概念です。すべてのサスペンド機能は、無制限のKotlinコルーチンで完全に使用できます。非同期スタイルの関数はフレームワークに依存します。
すべてのプロミス/フューチャーのフレームワークは、独自のプロミス/フューチャークラスを返す `async` のような関数や `await` のような関数も定義しなければなりません。

**パフォーマンス**を比較します。サスペンド関数は、呼び出しごとに最小限のオーバーヘッドを提供します。[実装の詳細](#実装の詳細)セクションでチェックアウトできます。
非同期スタイルの関数は、そのすべての中断機構に加えて、かなり重いプロミス/フューチャーの抽象化を維持する必要があります。
フューチャーのようなオブジェクトインスタンスは、非同期スタイルの関数呼び出しから常に返されなければならず、関数が非常に短く単純な場合でも最適化できません。
非同期スタイルは非常に細かい分解には適していません。

JVM/JSコードとの**相互運用性**を比較します。非同期スタイルの関数は、フューチャーのような抽象化のタイプを使用するJVM/JSコードとの相互運用性が向上します。JavaやJSでは、それは対応するフューチャーのようなオブジェクトを返す関数に過ぎません。サスペンド関数は、[継続渡しスタイル](#継続渡しスタイル)をネイティブにサポートしない言語からは奇妙に見えます。しかし、上の例では、任意のプロミス/フューチャーのフレームワークに対して、サスペンド関数を非同期スタイルの関数に変換するのが簡単であることがわかります。
したがって、Kotlinでサスペンド関数を一度書くだけで、`future{}` コルーチンビルダー関数を使ってコード1行で任意のプロミス/フューチャースタイルと相互運用することができます。

## 実装の詳細

このセクションではコルーチンの実装の詳細を垣間見ることができます。これらは[コルーチンの概要](#コルーチンの概要)セクションで説明されているビルディングブロックの背後に隠されていて、内部クラスやコード生成戦略は公開APIやABIの契約を破らない限りいつでも変更されることがあります。

### 継続渡しスタイル

サスペンド関数は、継続渡しスタイル（CPS）によって実装されます。
すべてのサスペンド関数とサスペンドラムダはそれが呼び出されたときに暗黙的に渡される追加の `Continuation` パラメータを持ちます。
`await` サスペンド関数の宣言が次のようになっていることを思い出してください。

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

ただし、実際の _実装_ では _CPS変換_ 後に次のシグネチャがあります。

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

その結果の型 `T` は、追加の継続パラメータの型引数の位置に移動しました。実装結果の型 `Any?` は、サスペンド関数の動作を表すように設計されています。サスペンド関数がコルーチンを _中断_ すると、特殊なマーカー値 `COROUTINE_SUSPENDED` が返されます。サスペンド関数がコルーチンを中断しないでコルーチンの実行を続けると、直接結果を返したり例外をスローしたりします。
このように、`await` 実装の `Any?` リターンタイプは、実際にはKotlinの型システムでは表現できない` COROUTINE_SUSPENDED`と `T`の和集合です。

サスペンド関数の実際の実装では、スタックフレーム内の継続を直接呼び出すことはできません。これは、長期実行のコルーチンでスタックオーバーフローが発生する可能性があるためです。
標準ライブラリの `suspendCoroutine` 関数は、継続の呼び出しを追跡することによってアプリケーション開発者からこの複雑さを隠し、継続が呼び出される方法と時間に関係なく、サスペンド関数の実際の実装規約に確実に適合します。

### 状態マシン

コルーチンを効率的に実装すること、つまりできるだけクラスとオブジェクトを少なくすることが重要です。多くの言語が _状態マシン_ としてそれらを実装し、Kotlinも同じことをしています。Kotlinの場合、このアプローチは、コンパイラが本体に任意の数の中断ポイントを持つ可能性のあるサスペンドラムダにつきクラスを1つだけ作成することになります。

主なアイデア。サスペンド関数は、状態が中断ポイントに対応する状態マシンにコンパイルされます。
例：2つの中断ポイントを持つサスペンドブロックを作成しましょう。
 
```kotlin
val a = a()
val y = foo(a).await() // 中断ポイント #1
b()
val z = bar(a, y).await() // 中断ポイント #2
c(z)
``` 

このコードブロックには3つの状態があります。

* 初期（中断ポイントの前）
* 最初の中断後
* 第2の中断ポイントの後

すべての状態は、このブロックの継続の1つのエントリポイントです（最初の継続は最初の行から続きます）。

このコードは、状態マシンを実装するメソッド、状態マシンの現在の状態を保持するフィールド、状態間で共有されるコルーチンのローカル変数のフィールドを持つ匿名クラスにコンパイルされます（コルーチンのクロージャのフィールドがあるかもしれませんが、この場合は空です）。
サスペンド関数の呼び出しのために継続渡しスタイルを使用する、上記のブロックの擬似Javaコードはawait次のとおりです。
これは、サスペンド関数 `await` を呼び出すために継続渡しスタイルを使用する、上記のブロックの擬似Javaコードです。
  
``` java
class <anonymous_for_state_machine> extends CoroutineImpl<...> implements Continuation<Object> {
    // 状態マシンの現在の状態
    int label = 0
    
    //コルーチンのローカル変数
    A a = null
    Y y = null
    
    void resume(Object data) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        //この呼び出しでdataは `null` になると予想される
        a = a()
        label = 1
        data = foo(a).await(this) // 'this' は継続として渡される
        if (data == COROUTINE_SUSPENDED) return // awaitが実行を中断した場合はリターンする
      L1:
        // 外部コードがawaitの結果をdataとして渡してコルーチンを再開した
        y = (Y) data
        b()
        label = 2
        data = bar(a, y).await(this) // 'this' は継続として渡される
        if (data == COROUTINE_SUSPENDED) return // awaitが実行を中断した場合はリターンする
      L2:
        // 外部コードがawaitの結果をdataとして渡してコルーチンを再開した
        Z z = (Z) data
        c(z)
        label = -1 // これ以上のステップは許されない
        return
    }          
}    
```  

例にはソースコードではなくバイトコードで何が起こるのかが示されているので、 `goto` 演算子とラベルがあることに注意してください。

コルーチンが起動したとき、`resume()`を呼び出します。`label` は `0` で、`L0` にジャンプしてからいくらかの作業をして、`label` を次の状態 `1` に設定します。`.await()` を呼び出し、コルーチンの実行が中断されたらリターンします。
実行を続行したいときには `resume()` をもう一度呼び出すと今度は `L1` へと進み、いくらか作業をして状態を `2` に設定して `.await()` を呼び出し、中断の場合は再びリターンします。
次回は `L3` から状態を `-1` に設定して続けます。これは「終わった、やるべきことはもうない」という意味です。

ループ内のサスペンドポイントは1つの状態しか生成しません。これはループも（条件付き）`goto` によって動作するからです。
 
```kotlin
var x = 0
while (x < 10) {
    x += nextNumber().await()
}
```

は、以下のように生成されます

``` java
class <anonymous_for_state_machine> extends CoroutineImpl<...> implements Continuation<Object> {
    // 状態マシンの現在の状態
    int label = 0
    
    // コルーチンのローカル変数
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
        data = nextNumber().await(this) // 'this' は継続として渡される
        if (data == COROUTINE_SUSPENDED) return // awaitが実行を中断した場合はリターンする
      L1:
        // 外部コードがawaitの結果をdataとして渡してコルーチンを再開した
        x += ((Integer) data).intValue()
        label = -1
        goto LOOP
      END:
        label = -1 // これ以上のステップは許されない
        return 
    }          
}    
```  

### サスペンド関数のコンパイル

サスペンド関数のコンパイルされたコードは、他のサスペンド関数をいつどのように呼び出すかによって異なります。
最も単純なケースでは、サスペンド関数は、他のサスペンド関数を _末尾の位置_ でのみ呼び出すため、 _末尾の呼び出し_ が行われます。
[サスペンド関数](#サスペンド関数)や[コールバックのラッピング](#コールバックのラッピング)のセクションで示すように、低レベルの同期プリミティブやコールバックのラップを実装するサスペンド関数の典型的なケースです。
これらの関数は、末尾で `suspendCoroutine` のような他のサスペンド関数を呼び出します。
これらは、通常の非サスペンド関数と同様にコンパイルされますが、唯一の例外は[CPS変換](#継続渡しスタイル)から得た暗黙の継続パラメーターが末尾呼び出しの次のサスペンド関数に渡されることです。

> 注：現行の実装では、`Unit` を返す関数は、他のサスペンド関数の呼び出しが末尾呼び出しとして認識されるように、明示的な `return` ステートメントを含める必要があります。

サスペンド呼び出しが末尾以外に現れる場合、コンパイラは対応するサスペンド関数用の[状態マシン](#状態マシン)を作成します。
サスペンド関数が呼び出されたときに作成され、完了すると破棄される状態マシンオブジェクトのインスタンスです。

> 注：将来のバージョンでは、このコンパイル戦略を最初の中断ポイントでのみステートマシンのインスタンスを作成ように最適化します。

この状態マシンオブジェクトは、末尾以外の位置での他のサスペンド関数の呼び出しの _完了の継続_ として機能します。
この状態マシンオブジェクトインスタンスは、関数が他のサスペンド関数に対して複数の呼び出しを行うときに更新され、再使用されます。
これを他の[非同期プログラミングスタイル](#非同期プログラミングスタイル)と比較します。非同期処理に続く各ステップは、通常、別々に新しく割り当てられたクロージャオブジェクトで実装されます。

### コルーチンの基礎

標準ライブラリーにおける `suspendCoroutine` サスペンド関数の実際の実装はKotlin自身で書かれており、ソースコードは標準ライブラリソースパッケージの一部として利用可能です。
コルーチンの安全で問題のない使用を提供するために、コルーチンの各中断の追加オブジェクトに状態マシンの実際の継続をラップします。
対応する非同期プリミティブのランタイムコストは、追加で割り当てられたオブジェクトのコストをはるかに上回るため、これは[非同期計算](#非同期計算)や[フューチャー](#フューチャー)のような真の非同期ユースケースにとってはまったく問題ありません。
しかし、ジェネレーターのユースケースでは、この追加コストは法外なものです。

標準ライブラリの `kotlin.coroutines.experimental.intrinsics` パッケージには、以下のシグネチャを持つ `suspendCoroutineOrReturn` という名前の関数が含まれています。

```kotlin
suspend fun <T> suspendCoroutineOrReturn(block: (Continuation<T>) -> Any?): T
```

これはサスペンド関数の[継続渡しスタイル](#継続渡しスタイル)への直接アクセスと、継続への _未チェック_ の参照を提供します。
`suspendCoroutineOrReturn` の利用者は、以下のCPS結果規約に全面的な責任を負いますが、その結果として少し良いパフォーマンスを得ます。
この慣習は通常 `buildSequence`/`yield` のようなコルーチンに対しては簡単にできますが、`suspendCoroutineOrReturn` の上に非同期 `await` のようなサスペンド関数を書こうとすると、`suspendCoroutine` の助けを借りずに正しく実装するのは**非常に難しい**ので**失望**しますし、これらの実装の試行でのエラーは、テストで見つけて再現しようとする試みを無視する典型的な[ハイゼンバグ](https://ja.wikipedia.org/wiki/%E7%89%B9%E7%95%B0%E3%81%AA%E3%83%90%E3%82%B0)です。

次のシグネチャを持つ `createCoroutineUnchecked` という関数もあります。

```kotlin
fun <T> (suspend () -> T).createCoroutineUnchecked(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutineUnchecked(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

それらは初期の継続への未チェックの参照を返します（追加のラッパーオブジェクトなしで）。
`createCoroutineUnchecked` による `buildSequence` の最適化バージョンを以下に示します。

```kotlin
fun <T> buildSequence(block: suspend SequenceBuilder<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutineUnchecked(receiver = this, completion = this)
    }
}
```

`suspendCoroutineOrReturn`による `yield` の最適化バージョンを以下に示します。
`yield` は常に中断されるので、対応するブロックは常に `COROUTINE SUSPENDED` を返します。

```kotlin
// ジェネレーターの実装
override suspend fun yield(value: T) {
    setNext(value)
    return suspendCoroutineOrReturn { cont ->
        nextStep = cont
        COROUTINE_SUSPENDED
    }
}
```

> [ここ](examples/sequence/buildSequenceOptimized.kt)で完全なコードを取得できます

`kotlin.coroutines.experimental.intrinsics` パッケージの内容は、誤って使用されるのを防ぐためIDEAのKotlinプラグインの自動補完から隠されています。上記の基礎的関数にアクセスするには、対応するimport文を手動で記述する必要があります。
 
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
 

