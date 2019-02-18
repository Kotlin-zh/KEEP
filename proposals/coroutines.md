# Kotlin 协程

* **类型**：设计提案
* **作者**：Andrey Breslav, Roman Elizarov
* **贡献者**： Vladimir Reshetnikov, Stanislav Erokhin, Ilya Ryzhenkov, Denis Zharkov
* **状态**：从 Kotlin 1.3 开始稳定，在 Kotlin 1.1-1.2 中为实验性

## 摘要

本文是对 Kotlin 协程的描述。这一概念通常被认为与下列内容有关，或部分涵盖它们：

- generators/yield
- async/await
- composable/delimited сontinuations

设计目标：

- 不依赖 Future 之类复杂的库提供的特定实现；
- 同时涵盖 “async/await” 用例和“生成器代码块”；
- 使 Kotlin 协程能包装各种现有的异步 API 
 （如 Java NIO、各种 Future 的实现等）；

## 目录

* [用例](#用例)
  * [异步计算](#异步计算)
  * [Future](#Future)
  * [生成器](#生成器)
  * [异步 UI](#异步-UI)
  * [其他用例](#其他用例)
* [协程概述](#协程概述)
  * [术语](#术语)
  * [续体接口](#续体接口)
  * [挂起函数](#挂起函数)
  * [协程构建器](#协程构建器)
  * [协程上下文](#协程上下文)
  * [续体拦截器](#续体拦截器)
  * [受限挂起](#受限挂起)
* [实现细节](#实现细节)
  * [续体传递风格](#续体传递风格)
  * [状态机](#状态机)
  * [编译挂起函数](#编译挂起函数)
  * [协程内建函数](#协程内建函数)
* [附录](#附录)
  * [资源管理与垃圾收集](#资源管理与垃圾收集)
  * [并发和线程](#并发和线程)
  * [异步编程风格](#异步编程风格)
  * [包装回调](#包装回调)
  * [构建 Future](#构建-Future)
  * [非阻塞休眠](#非阻塞休眠)
  * [协作式单线程多任务](#协作式单线程多任务)
  * [异步序列](#异步序列)
  * [通道](#通道)
  * [互斥](#互斥)
  * [从实验性协程移植](#从实验性协程移植)
  * [参考](#参考)
  * [反馈](#反馈)
* [版本历史](#版本历史)
  * [3.3 版本中的改动](#3.3-版本中的改动)
  * [3.2 版本中的改动](#3.2-版本中的改动)
  * [3.1 版本中的改动](#3.1-版本中的改动)
  * [3 版本中的改动](#3-版本中的改动)
  * [2 版本中的改动](#2-版本中的改动)

## 用例

协程可以被视作*可挂起的计算*的实例。即，可以在某些<!--
-->点上挂起，稍后在另一个线程上恢复执行。协程相互调用<!--
-->（互相传递数据），即可形成<!--
-->协作式多任务处理机制。

### 异步计算

最能描述协程功能的用例是异步计算<!--
-->（在 C# 及其他语言中通过 `async`/`await` 实现）。<!--
-->让我们来看看如何通过回调完成这样的计算。不妨以<!--
-->异步 I/O 为例（下面的 API 经过简化）：

```kotlin
// 异步读数据到 `buf`，完成后执行 lambda 表达式
inChannel.read(buf) {
    // 这个 lambda 表达式会在读完后执行
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)
    
    // 异步从 `buf` 写数据, 完成后执行 lambda 表达式
    outChannel.write(buf) {
        // 这个 lambda 表达式会在写完后执行
        ...
        ...
        outFile.close()          
    }
}
```

注意，我们在回调内部有一个回调，虽然这能节省很多没有意义的代码（例如，没有<!--
-->必要将  `buf ` 参数显式传递给回调，它们只是将它视为闭包的一部分），但缩进<!--
-->级别每次都在增长，而且只要嵌套超<!--
-->过一层，大家都知道会产生多少麻烦（谷歌搜索“回调地狱”，看看 JavaScript 迫害了多少人）。

同样的计算可以直截了当地表达为协程（前提是有一个合适的库，<!--
-->使 IO 应用程序接口适配协程的需求）：

```kotlin
launch {
    // 异步读时挂起
    val bytesRead = inChannel.aRead(buf) 
    // 读完成后才执行这一行
    ...
    ...
    process(buf, bytesRead)
    // 异步写时挂起
    outChannel.aWrite(buf)
    // 写完成后才执行这一行
    ...
    ...
    outFile.close()
}
```

这里的 `aRead()` 和 `aWrite()` 是特殊的*挂起函数* —— 它们可以*挂起*代码执行<!--
-->（这并不意味着阻塞正在运行它的线程），并在调用完成时*恢复*。<!--
-->如果我们眯起眼睛，可以想象所有在 `aRead()` 之后的代码已经被包装成一个
lambda 表达式并作为回调传递给 `aRead()`，对 `aWrite()` 也是如此，<!--
-->我们就可以看到这段代码和上面的相同，可读性却更强。

我们的明确目标是以一种非常通用的方式支持协程，所以在此示例中，<!--
-->`launch{}`、`.aRead()` 和 `.aWrite()` 只是适应协程工作的**库函数**；<!--
-->`launch` 是*协程构建器* —— 它创建并启动协程，<!--
-->而 `aRead()` 与 `aWrite()` 作为特殊的<!--
-->*挂起函数*，它隐式地接受<!--
-->*续体*（续体就是一般的回调）。

> 关于 `launch{}` 的示例代码在[协程构建器](#协程构建器)一节，<!--
-->关于 `aRead()` 的示例代码在[包装回调](#包装回调)一节。

注意，显式传入的回调要在循环中异步调用通常非常棘手，<!--
-->但在协程中这不过是稀松平常的小事：

```kotlin
launch {
    while (true) {
        // 异步读时挂起
        val bytesRead = inFile.aRead(buf)
        // 读完继续执行
        if (bytesRead == -1) break
        ...
        process(buf, bytesRead)
        // 异步写时挂起
        outFile.aWrite(buf) 
        // 写完继续执行
        ...
    }
}
```

可想而知，在协程中处理异常也会稍微方便一些。

### Future

还有另一种表达异步计算的方式：通过 future（也称为 promise 或 deferred）。<!--
-->我们将在在示例中使用一个虚构的应用程序接口，将叠加层应用于图像：

```kotlin
val future = runAfterBoth(
    loadImageAsync("...original..."), // 创建一个 Future
    loadImageAsync("...overlay...")   // 创建一个 Future
) {
    original, overlay ->
    ...
    applyOverlay(original, overlay)
}
```

使用协程，可以写成：

```kotlin
val future = future {
    val original = loadImageAsync("...original...") // 创建一个 Future
    val overlay = loadImageAsync("...overlay...")   // 创建一个 Future
    ...
    // 等待图片加载时挂起
    // 二者都加载完后执行 `applyOverlay(...)`
    applyOverlay(original.await(), overlay.await())
}
```

> 关于 `future{}` 的示例代码在[构建 Future](#构建-Future) 一节，<!--
-->关于 `await()` 的示例代码在[挂起函数](#挂起函数)一节。

同样，协程通过更少的缩进以及更自然的组合逻辑（以及异常处理，这里没有显示），<!--
-->而且没有使用专门的关键字（比如 C#、JS 以及其他语言中的 `async` 和 `await`）来支持 future：<!--
-->`future{}` 和 `.await()` 都只是库函数而已。

### 生成器

协程的另一个典型用例是延时计算序列（在 C#、Python <!--
-->和很多其他语言中通过 `yield`  实现）。这样的序列可以由看似顺序的代码生成，但在运行时只<!--
-->计算所请求的元素：

```kotlin
// 推断出类型为 Sequence<Int>
val fibonacci = sequence {
    yield(1) // 斐波那契数列的首项
    var cur = 1
    var next = 1
    while (true) {
        yield(next) // 斐波那契数列的下一项
        val tmp = cur + next
        cur = next
        next = tmp
    }
}
```

代码创建了一个表示[斐波那契数列](https://zh.wikipedia.org/wiki/斐波那契数列)的延迟序列，<!--
-->它可以是无限长的<!--
-->（类似 [Haskell 的无限长列表](http://www.techrepublic.com/article/infinite-list-tricks-in-haskell/)）。<!--
-->我们可以只计算其中一部分，例如，通过 `take()`：

```kotlin
println(fibonacci.take(10).joinToString())
```

> 这会打印出 `1, 1, 2, 3, 5, 8, 13, 21, 34, 55`。<!--
-->你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/fibonacci.kt)试一下。

生成器的优势在于支持任意的控制流，包括但不限于 `while`、<!--
-->`if`、`try`/`catch`/`finally`：

```kotlin
val seq = sequence {
    yield(firstItem) // 挂起点
    
    for (item in input) {
        if (!item.isValid()) break // 不再生成项
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // 挂起点        
    }
    
    try {
        yield(lastItem()) // 挂起点
    }
    finally {
        // 一些收尾代码
    }
} 
```

> 关于 `sequence{}` 和 `yield()` 的示例代码在<!--
-->[限定挂起](#限定挂起)一节。

注意，这种方法还允许把 `yieldAll(sequence)` 表示为库函数<!--
-->（像 `sequence{}` 和 `yield()` 那样），这能简化延时序列的连接操作，<!--
-->并提升了性能。

### 异步 UI

典型的 UI 应用程序只有一个事件调度线程，所有 UI 操作都发生在这个线程上。<!--
-->通常不允许在其他线程修改 UI 状态。所有 UI 库都提供<!--
-->某种原语，以将操作转移回 UI 线程中执行。例如，Swing 的
[`SwingUtilities.invokeLater`](https://docs.oracle.com/javase/8/docs/api/javax/swing/SwingUtilities.html#invokeLater-java.lang.Runnable-)，<!--
-->JavaFX 的
[`Platform.runLater`](https://docs.oracle.com/javase/8/javafx/api/javafx/application/Platform.html#runLater-java.lang.Runnable-)，<!--
-->Android 的
[`Activity.runOnUiThread`](https://developer.android.com/reference/android/app/Activity.html#runOnUiThread(java.lang.Runnable))
等等。<!--
-->下面是一个典型的 Swing 应用程序的代码片段，它执行一些异步<!--
-->操作，然后在 UI 中显示其结果：

```kotlin
makeAsyncRequest {
    // 异步请求完成时执行这个 lambda 表达式
    result, exception ->
    
    if (exception == null) {
        // 在 UI 线程显示结果
        SwingUtilities.invokeLater {
            display(result)   
        }
    } else {
       // 异常处理
    }
}
```

这很像我们之前在[异步计算](#异步计算)用例见过的回调地狱，<!--
-->所以也能通过协程优雅地解决：

```kotlin
launch(Swing) {
    try {
        // 执行异步请求时挂起
        val result = makeRequest()
        // 在 UI 上显示结果，Swing 上下文保证了我们位于事件调度线程上
        display(result)
    } catch (exception: Throwable) {
        // 异常处理
    }
}
```

> `Swing` 上下文的示例代码在[续体拦截器](#续体拦截器)一节。

所有的异常处理也都可以使用原生的的语法结构执行。

### 其他用例

协程可以覆盖更多用例，比如下面这些：

* 基于通道的并发（就是 Go 协程和通道）；
* 基于 Actor 模式的并发；
* 偶尔需要用户交互的后台进程，例如显示模式对话框；
* 通信协议：将每个参与者实现为一个序列，而不是状态机；
* Web 应用程序工作流：注册用户、验证电子邮件、登录<!--
  -->（挂起的协程可以序列化并存储在数据库中）。

## 协程概述

本部分概述了支持编写协程的语言机制和<!--
-->管理其语义的标准库。

### 术语

* *协程*——*可挂起计算*的*实例*。它在概念上类似于线程，在这个意义上，<!--
  -->它需要一个代码块运行，并具有类似的生命周期 —— 它可以被*创建*和*启动*，但它不绑定到<!--
  -->任何特定的线程。它可以在一个线程中*挂起*其执行，并在另一个线程中*恢复*。<!--
  -->而且，像 future 或 promise 那样，它在*完结*时可能伴随着某种结果（值或异常）。

* *挂起函数*—— `suspend` 修饰符标记的函数。它可能会通过调用其他挂起函数*挂起*执行代码，<!--
  -->而不阻塞当前执行线程。挂起函数<!--
  -->不能在常规代码中被调用，只能在其他挂起函数或挂起 lambda 表达式中（见下方）。<!--
  -->例如，[用例](#用例)所示的 `.await()` 和 `yield()` 是<!--
  -->在库中定义的挂起函数。标准库提供了原始的挂起函数，用于定义<!--
  -->其他所有挂起函数。

* *挂起 lambda 表达式*——必须在协程中运行的代码块。<!--
  -->它看起来很像一个普通的 [lambda 表达式](https://kotlinlang.org/docs/reference/lambdas.html)，<!--
  -->但它的函数类型被 `suspend` 修饰符标记。<!--
  -->就像常规 lambda 表达式是匿名局部函数的短语法形式一样，<!--
  -->挂起 lambda 表达式是匿名挂起函数的短语法形式。它可能会通过调用其他挂起函数*挂起*执行代码，<!--
  -->而不阻塞当前执行线程。<!--
  -->例如，[用例](#用例)所示的跟在 `launch` , `future` , 和 `BuildSequence` 函数后面花括号里的代码块<!--
  -->就是挂起 lambda 表达式。

  > 注意：挂起 lambda 表达式可以在其代码的任意位置调用挂起函数，只要这个位置能编写从这个 lambda 表达式<!--
  -->[非局部](https://kotlinlang.org/docs/reference/returns.html) `return` 的语句。<!--
  -->也就是说，可以在像 [`apply{}` 代码块](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html)<!--
  -->这样的内联 lambda 表达式中调用挂起函数，<!--
  -->但在 `noinline` 和 `crossinline` 修饰的 lambda 表达式中就不行。<!--
  -->*挂起*会被视作是一种特殊的非局部控制转移。

* *挂起函数类型*——表示挂起函数和挂起 lambda 表达式的函数类型。它就像<!--
  -->一个一般的[函数类型](https://kotlinlang.org/docs/reference/lambdas.html#function-types)，<!--
  -->但具有 `suspend` 修饰符。举个例子，`suspend () -> Int` 是<!--
  -->一个没有参数、返回 `Int` 的挂起函数的函数类型。一个声明为 `suspend fun foo()  : Int` 的挂起函数<!--
  -->符合上述函数类型。

* *协程构建器*——使用一些挂起 lambda 表达式作为参数来创建一个协程的函数，<!--
  -->并且可选地，还提供某种形式以访问协程的结果。例如，[用例](#用例)中的 `launch{}`、`future{}` <!--
  -->以及 `sequence{}` 就是协程构建器。<!--
  -->标准库提供了用于定义其他所有协程构建器所使用的原始协程构建器。

  > 注意：一些语言通过对特定方法的硬编码支持协程的创建、启动、<!--
  -->定义其执行的方式以及结果的表示方式。例如，`generate` *关键字*可以定义<!--
  -->返回某种可迭代对象的协程，而 `async` *关键字*可以定义<!--
  -->返回某种约定或任务的协程。Kotlin 没有关键字或修饰符来定义和启动协程。<!--
  -->协程构建器只是库中定义的简单的函数。<!--
  -->其他语言中以方法体形式定义的协程，<!--
  -->在 Kotlin 中，这样的方法通常是具有表达式方法体的普通方法，<!--
  -->方法体的内容是调用一个库中定义的、最后一个参数是挂起 lambda 表达式的协程构建器：

  ```kotlin
  fun doSomethingAsync() = async { ... }
  ```

* *挂起点*——协程执行过程中*可能被挂起*的位置。<!--
  -->从语法上说，挂起点是对一个挂起函数的调用，但*实际*<!--
  -->的挂起在挂起函数调用了标准库中的原始挂起函数时发生。

* *续体*——是挂起的协程在挂起点时的状态。它在概念上代表<!--
  -->它在挂起点之后的剩余应执行的代码。例如：

  ```kotlin
  sequence {
      for (i in 1..10) yield(i * i)
      println("over")
  }  
  ```

  这里，每次调用挂起函数 `yield()`时，协程都会挂起，<!--
  -->*其执行的剩余部分*被视作续体，所以有 10 个<!--
  -->续体：循环运行第一次后，`i=2`，挂起；循环运行第二次后，`i=3`，挂起……<!--
  -->最后一次打印“over”并完结协程。已经*创建*，但尚未<!--
  -->*启动*的协程，由它的*初始续体*表示，这由它的整个执行组成，<!--
  -->类型为 `Continuation<Unit> `。

如上所述，驱动协程的要求之一是灵活性：<!--
-->我们希望能够支持许多现有的异步应用程序接口以及其他用例，并尽量减少<!--
-->硬编码到编译器中的部分。因此，编译器只负责支持<!--
-->挂起函数、挂起 lambda 表达式以及相应的挂起函数类型。<!--
-->标准库中的原语很少，其余的则留给应用程序库。

### 续体接口

这是标准库中接口 <!--
-->[`Continuation`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/index.html) 的定义<!--
-->（位于 `kotlinx.coroutines` 包），代表了一个通用的回调：

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resumeWith(result: Result<T>)
}
```

上下文将在[协程上下文](#协程上下文)一节中详细介绍，表示与协程关联的任意<!--
-->用户定义上下文。`resumeWIth` 函数是一个*完结*<!--
-->回调，用于报告协程完结时成功（带有值）或失败（带有异常）的结果。

为了方便，包里还定义了两个扩展函数：

```kotlin
fun <T> Continuation<T>.resume(value: T)
fun <T> Continuation<T>.resumeWithException(exception: Throwable)
```

### 挂起函数

一个典型的*挂起函数*的某种实现，例如 `.await()`，看起来是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) // 这个 future 正常完结了
                cont.resume(result)
            else // 这个 future 因为异常而完结了
                cont.resumeWithException(exception)
        }
    }
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/future/await.kt)找到代码。<!--
-->注意：这个简单的实现只要 future 不完结就会永远挂起协程。<!--
-->[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实际实现<!--
-->还支持取消。

`suspend` 修饰符表明这个函数可以挂起协程的执行。<!--
-->这个特殊的函数被定义为类型 `CompletableFuture<T>` 的<!--
-->[扩展函数](https://kotlinlang.org/docs/reference/extensions.html)，<!--
-->以便使用它时能自然按照<!--
-->与实际执行顺序相对应的从左到右的顺序读取:

```kotlin
doSomethingAsync(...).await()
```

`suspend` 修饰符可以用于任何函数：顶层函数、扩展函数、成员函数、<!--
-->局部函数或操作符函数。

> 属性的取值器和设值器、构造函数以及某些操作符函数<!--
-->（也就是 `getValue`，`setValue`，`provideDelegate`，`get`，`set` 以及 `equals`）不能带有 `suspend` 修饰符。<!--
-->这些限制将来可能会被移除。

挂起函数可以调用任何常规函数，但要真正挂起执行，必须<!--
-->调用一些其他的挂起函数。特别是，这个 `await` 实现调用了<!--
-->在标准库中定义的顶层挂起函数 <!--
-->[`suspendCoroutine`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html)<!--
-->（位于 `kotlinx.coroutines` 包）：

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

当 `suspendCoroutine` 在一个协程中被调用时（它*只*可能在协程中<!--
-->被调用，因为它是一个挂起函数），它捕获了协程的执行状态<!--
-->到一个*续体*实例，然后将其作为参数传递给指定的 `block`。<!--
-->为了恢复协程的执行，代码块需要在该线程或稍后在其他某个线程中调用 `continuation.resumeWith()`<!--
-->（直接调用 `continuation.resume()` 或 `continuation.resumeWithException()` 扩展）。<!--
--><!--
-->当 `suspendCoroutine` 代码块没有调用 `resumeWith` 就返回时，会发生*实际*的协程挂起。<!--
-->如果续体还未从代码块返回就直接被恢复，<!--
-->协程就不被看作已经暂停又继续执行。

传给 `continuation.resumeWith()` 的值作为调用 `suspendCoroutine` 的结果，<!--
-->进一步成为 `.await()` 的结果。

不允许多次恢复同一个协程，并会产生 `IllegalStateException`。 

> 注意：这正是 Kotlin 协程与像 Scheme 这样的函数式语言中的顶层限定续体<!--
-->或 Haskell 中的续体单子的关键区别。我们选择仅支持续体恢复一次，<!--
-->完全是出于实用主义考虑，因为所有这些预期的[用例](#用例)都不需要多重续体。<!--
-->然而，还是可以在单独的库中实现多重续体，<!--
-->通过底层的所谓[协程内建函数](#协程内建函数)复制续体中<!--
-->捕获的协程状态，然后就可以从这个副本再次恢复协程。

### 协程构建器

挂起函数不能从常规函数中调用，所以标准库提供了<!--
-->用于在常规非挂起作用域中启动协程执行的函数。这是<!--
-->简化的*协程构建器* `launch` 的实现：

```kotlin
fun launch(context: CoroutineContext = EmptyCoroutineContext, block: suspend () -> Unit) =
    block.startCoroutine(Continuation(context) { result ->
        result.onFailure { exception ->
            val currentThread = Thread.currentThread()
            currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
        }
    })
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/run/launch.kt)获取代码。

这个实现使用了 [`Continuation(context) { ... }`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html) <!--
-->函数（来自 `kotlin.coroutines` 包），该函数提供了<!--
-->实现 `Continuation` 接口的快捷方式，该接口具有其 `context` 的给定值和
`resumeWith` 函数的主体。这个续体作为*完结续体*被传递给
[`block.startCoroutine(...)`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html) 扩展函数<!--
-->（来自 `kotlin.coroutines` 包）。

协程在完结时将调用其*完结续体*。其 `resumeWith`
函数将在协程因成功或失败而*完结*时调用。<!--
-->因为 `launch` 是那种“即发即弃”式的<!--
-->协程，它被定义成返回 `Unit` 的挂起函数，实际上是忽略了
`resume` 函数的结果。如果协程因异常完结，<!--
-->当前线程的未捕获异常句柄将用于报告这个异常。

> 注意：这个简单实现返回了 `Unit` 并且根本不提供对协程状态的任何访问。<!--
-->实际上在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实现要更加<!--
-->复杂，因为它返回了一个代表这个协程的 `Job` 接口的实例，而且可以被取消。

上下文在[协程上下文](#协程上下文)一节中详细介绍。<!--
-->`startCoroutine` 在标准库中定义为无参数和<!--
-->单参数的挂起函数类型的扩展函数：

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
fun <R, T> (suspend  R.() -> T).startCoroutine(receiver: R, completion: Continuation<T>)
```

`startCoroutine` 创建协程并在当前线程中立刻启动执行（但请参阅下面的备注），<!--
-->直到第一个*挂起点*时返回。<!--
-->挂起点是协程中某个[挂起函数](#挂起函数)的调用，<!--
-->由相应的挂起函数的代码来定义协程恢复的时机和方式。

> 注意：续体拦截器（来自上下文）在[后文](#续体拦截器)中会提到，它能够<!--
-->将协程的执行，*包括*其初始续体的执行，调度到另一个线程中。

### 协程上下文

协程上下文是一组可以附加到协程中的持久化用户定义对象。<!--
-->它可以包括负责协程线程策略的对象，日志，关于协程执行的安全性和事务方面的对象，<!--
-->协程的标识和名称等等。下面是协程及其上下文的简单认识模型。<!--
-->把协程看作一个轻量线程。在这种情况下，协程上下文就像是一组线程局部变量。<!--
-->不同之处在线程局部变量是可变的，协程上下文是不可变的，<!--
-->但对于协程这并不是一个严重的限制，因为他们是如此轻量<!--
-->以至于当需要改变上下文时可以很容易地开启一个新的协程。

标准库没有包含上下文的任何具体实现，<!--
-->但是有接口和抽象类，以便以*可组合*的方式<!--
-->在库中定义所有这些方面，因此来自不同库的各个方面可以<!--
-->在同一个上下文中和平共存。

从概念上讲，协程上下文是一组索引元素，其中每个元素有唯一的键。<!--
-->它是 set 与 map 的混合体。它的元素有像在 map 中的那样的键，但它的键直接与元素关联，<!--
-->更像是 set。标准库定义了
[`CoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html) 的最小接口<!--
-->（位于 `kotlinx.coroutines` 包）：

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

`CoroutineContext` 本身支持四种核心操作：

* 操作符 [`get`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/get.html) <!--
  -->支持通过给定键类型安全地访问元素。可以使用 `[..]` 写法，<!--
  -->解释见 [Kotlin 操作符重载](https://kotlinlang.org/docs/reference/operator-overloading.html)。
* 函数 [`fold`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/fold.html) <!--
  -->类似于标准库中 [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) <!--
  -->扩展函数，提供迭代上下文中所有元素的方法。
* 操作符 [`plus`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/plus.html) <!--
  -->类似于标准库的 [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html) <!--
  -->扩展函数，返回两个上下文的组合, 同时加号右边的元素会替换掉<!--
  -->加号左边具有相同键的元素。
* 函数 [`minusKey`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/minus-key.html) <!--
  -->返回不包含指定键的上下文。

协程上下文的一个 [`Element`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-element/index.html) <!--
-->就是上下文本身。<!--
-->那是仅有这一个元素的上下文单例。<!--
-->这样就可以通过获取库定义的协程上下文元素并使用 `+` 连接它们，<!--
-->来创建一个复合上下文。例如，如果一个库定义的 `auth` 元素带着用户授权信息，<!--
-->而另一些库定义了带有一些带有上下文执行信息的 `threadPool` 对象，<!--
-->你就可以使用[协程构建器](#协程构建器) `launch{}` 使用组合上下文使用
`launch(auth + CommonPool){...}` 调用。

> 注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 提供了几个上下文元素，<!--
-->包括用于在一个共享后台线程池中调度协程的 `Dispatchers.Default` 对象。

标准库提供
[`EmptyCoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html)
——一个不包含任何元素的（空的）`CoroutineContext` 实例。

所有第三方协程元素应该继承标准库的
[`AbstractCoroutineContextElement`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-abstract-coroutine-context-element/index.html) <!--
-->类（位于 `kotlinx.coroutines` 包）。<!--
-->要在库中定义上下文元素，建议使用以下风格。<!--
-->以下示例展示了存储当前用户名的假设授权上下文元素：

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

> 可以在这里找到[示例](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/auth.kt)。

将上下文的 `Key` 定义为相应元素类的伴生对象能够流畅<!--
-->访问上下文中的相应元素。这是一个假想的的挂起函数实现，<!--
-->它需要检查当前用户名：

```kotlin
suspend fun doSomething() {
    val currentUser = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
    // 做一些用户指定的事
}
```

它使用了 [`coroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html) <!--
-->顶层属性（位于 `kotlinx.coroutines` 包），<!--
-->以在挂起函数中检索当前协程的上下文。

### 续体拦截器

让我们回想一下[异步 UI](#异步-UI)用例。异步 UI 应用程序必须保证<!--
-->协程程序体始终在 UI 线程中执行，尽管事实上各种挂起函数<!--
-->是在任意的线程中恢复协程执行。这是使用*续体拦截器*完成的。<!--
-->首先，我们要充分了解协程的生命周期。思考一下这个用了<!--
-->[协程构建器](#协程构建器) `launch{}` 的代码片段：

```kotlin
launch(CommonPool) {
    initialCode() // 执行初始化代码
    f1.await() // 挂起点 #1
    block1() // 执行 #1
    f2.await() // 挂起点 #2
    block2() // 执行 #2
}
```

协程从 `initialCode` 开始执行，直到第一个挂起点。在挂起点时，<!--
-->协程*挂起*，一段时间后按照相应挂起函数的定义，协程*恢复*并执行
`block1`，接着再次挂起又恢复后执行 `block2`，在此之后协程*完结*了。

续体拦截器可以选择拦截并包装<!--
-->与 `initialCode`，`block1` 和 `block2` 执行相对应的、从它们恢复的位置到下一个挂起点之间的续体。<!--
-->协程的初始化代码被视作是<!--
-->由协程的*初始续体*恢复得来。标准库提供了
[`ContinuationInterceptor`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html)
接口（位于 `kotlinx.coroutines` 包）：

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    fun releaseInterceptedContinuation(continuation: Continuation<*>)
}
```

`interceptContinuation` 函数包装了协程的续体。每当协程被挂起时，<!--
-->协程框架用下面这行代码包装实际后续恢复的 `continuation`：


```kotlin
val intercepted = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```

协程框架为每个实际的续体实例缓存拦截过的续体，并且在不再需要时<!--
-->调用 `releaseInterceptedContinuation(intercepted)`。<!--
-->想了解更多细节请参阅[实现细节](#实现细节)一节。

> 注意，像 `await` 这样的挂起函数实际上不一定会挂起协程的执行。<!--
  -->例如，[挂起函数](#挂起函数)一节所展现的 `await` 实现<!--
  -->在 future 已经完结的情况下就不会使协程真正挂起（在这种情况下 `resume` 会立刻被调用，<!--
  -->协程的执行并没有被挂起）。只有协程执行中真正被挂起时，续体才会被拦截，<!--
  -->即 `suspendCoroutine` 块返回而不调用 `resume`。


让我们来看看 `Swing` 拦截器的具体示例代码，它将执行调度到
Swing UI 事件调度线程上。我们先来定义一个包装类 `SwingContinuation`，<!--
-->它调用 `SwingUtilities.invokeLater`，把续体调度到 Swing 事件调度线程：

```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> {
    override val context: CoroutineContext = cont.context
    
    override fun resumeWith(result: Result<T>) {
        SwingUtilities.invokeLater { cont.resumeWith(result) }
    }
}
```

然后定义 `Swing` 对象并实现 `ContinuationInterceptor` 接口，用作对应的上下文元素：


```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/swing.kt)获得这部分代码。<!--
  -->注意：`Swing` 对象在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) <!--
  -->中的实际实现还支持了协程调试功能，提供对当前协程的标识符的访问和显示，<!--
  -->标识符用运行协程的线程表示。

现在，可以用带有 `Swing` 参数的[协程构建器](#协程构建器) `launch{}` 来<!--
-->执行完全运行在 Swing 事件调度线程中的协程：

```kotlin
launch(Swing) {
  // 这里的代码可以挂起，但总是恢复在 Swing 事件调度线程上
}
```

> 在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中，Swing 上下文的实际实现<!--
  -->更加复杂，因为它还要集成库的计时和调试工具。

### 限定挂起

为了实现[生成器](#生成器)用例中的 `sequence{}` 和 `yield()`，需要另一类协程构建器和挂起函数。<!--
-->以下是协程构建器 `sequence{}` 的示例代码：

```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

它使用了标准库中类似于 `startCoroutine`（解释见[协程构建器](#协程构建器)一节）的另一个原语
[`createCoroutine`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/create-coroutine.html)。<!--
--><!--
-->不同点在于它*创建*一个协程，但并*不*启动协程，<!--
-->而是返回表示协程的*初始续体*作为 `Continuation<Unit>` 的引用：

```kotlin
fun <T> (suspend () -> T).createCoroutine(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutine(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

另一个不同点是传递给构建器的*挂起 lambda 表达式* `block` 是<!--
-->具有 `SequenceScope<T>` 接收者的<!--
-->[扩展 lambda 表达式](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver)。<!--
-->`SequenceScope<T>` 接口提供了生成器代码块的*作用域*，其在库中定义如下：


```kotlin
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

为了避免生成多个对象，`sequence{}` 实现中定义了 `SequenceCoroutine<T>` 类，<!--
-->它同时实现了 `SequenceScope<T>` 与 `Continuation<Unit>`，因此它可以同时作为 <!--
-->`createCoroutine` 的 `receiver` 参数与 `completion` 续体参数。<!--
-->下面展示了 `SequenceCoroutine<T>` 的一种简单实现：

```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceScope<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // 实现抽象迭代器
    override fun computeNext() { nextStep.resume(Unit) }

    // 实现续体
    override val context: CoroutineContext get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Unit>) {
        result.getOrThrow() // 错误则退出
        done()
    }

    // 实现生成器
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequence.kt)获得该代码。<!--
  -->注意，标准库提供了
  [`sequence`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/sequence.html) 函数开箱即用的优化实现<!--
  -->（位于 `kotlinx.coroutines` 包），<!--
  -->而且还具有对 [`yieldAll`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence-scope/yield-all.html) 函数的额外支持。

> `sequence` 的实际代码使用了实验性的 `BuilderInference` 特性以支持<!--
  -->[生成器](#生成器)一节中使用的不用显式指定序列类型参数 `T` 的 `fibonacci` 声明。<!--
  -->相反，其类型是从传递给 `yield` 的参数类型推断得来的。

`yield` 的实现中使用了 `suspendCoroutine` [挂起函数](#挂起函数)来挂起。<!--
-->协程并捕获其续体。续体保存在 `nextStep` 中，<!--
-->并在调用 `computeNext` 时恢复。

然而，之前展示的 `sequence{}` 和 `yield()`，其续体并不能被任意的挂起函数<!--
-->在各自的作用域里捕获。它们*同步*地工作。<!--
-->它们需要对如何捕获续体、<!--
-->在何处存储续体和何时恢复续体保持绝对的控制。它们形成了*限定挂起域*。<!--
-->对挂起的限定作用由作用域类或接口上的 `RestrictSuspension` 注解提供，<!--
-->在上面的示例中这个作用域接口是 `SequenceScope`：

```kotlin
@RestrictsSuspension
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

这个注解对能用在 `sequence{}` 域或其他类似的同步协程构建器中的挂起函数有一定的限制。<!--
--><!--
-->那些扩展*限定性挂起域*类或接口<!--
-->（以 `@RestrictsSuspension` 标记）的<!--
-->挂起 lambda 表达式或函数称作*限定性挂起函数*。<!--
-->限定性挂起函数只接受来自同一个限定挂起域实例的的成员或扩展挂起函数作为参数。


回到这个例子，这意味着 <!--
-->`SequenceScope` 作用域的内扩展 lambda 表达式不能调用 `suspendContinuation` 或其他<!--
-->通用挂起函数。要挂起 `sequence` 协程的执行，最终必须通过调用 <!--
-->`SequenceScope.yield`。`yield` 本身被实现为 `SequenceScope` <!--
-->实现的成员函数，对其内部不作任何限制（只有*扩展*挂起 lambda 表达式和函数是限定的）。

对于像 `sequence` 这样的限定性协程构建器，支持任意上下文是没有意义的，<!--
-->因为其作用类或接口（比如这个例子里的 `SequenceScope`）已经占用了上下文能提供的服务，因此限定性<!--
-->协程只能使用 `EmptyCoroutineContext` 作为上下文，`SequenceCouroutine` 的取值器实现<!--
-->也会返回这个。尝试创建上下文不是 `EmptyCoroutineSContext` 的限定性协程<!--
-->会引发 `IllegalArgumentException`。

## 实现细节

本节展现了协程实现细节的冰山一角。它们隐藏在<!--
-->[协程概述](#协程概述)部分解释的构建代码块背后，<!--
-->内部类和代码的生成策略随时可能变化，<!--
-->只要不打破公共应用程序接口和应用程序二进制接口的约定。

### 续体传递风格

挂起函数通过 Continuation-Passing-Style (CPS) 实现。<!--
-->每个挂起函数和挂起 lambda 表达式都有一个附加的 `Continuation` 参数，<!--
-->在调用时隐式传入。回想一下，<!--
-->[`await` 挂起函数](#挂起函数)的声明是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

然而在 *CPS 变换*之后，它的实际*实现*具有以下签名：

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

其返回类型 `T` 移到了附加的续体参数的类型参数位置。<!--
-->实现中的返回值类型 `Any?` 被设计用于表示挂起函数的动作。<!--
-->当挂起函数*挂起*协程时，函数返回一个特别的标识值
`COROUTINE_SUSPENDED`（更多细节参考[`协程内建函数`](#协程内建函数)一节）。<!--
-->如果一个挂起函数没有挂起协程，<!--
-->协程继续执行时，它直接返回一个结果或者抛出一个异常。<!--
-->这样，`await` 函数实现中的返回值类型 `Any?` 实际上是 `T` 和 `COROUTINE_SUSPENDED` 的联合类型，<!--
-->这并不能在 Kotlin 的类型系统中表示出来。

挂起函数的实现实际上不允许直接调用其栈帧中的续体，<!--
-->因为在长时间运行的协程中这可能导致栈溢出。标准库中的 `suspendCoroutine` 函数<!--
-->通过追踪续体的调用对应用开发者隐藏这种复杂性，<!--
-->并确保无论续体在何时怎样调用，<!--
-->都与挂起函数的实际实现具有一致性的。

### 状态机

协程实现的性能至关重要，这需要尽可能少地创建类和对象。<!--
-->许多语言通过*状态机*实现，Kotlin 也是这样做的。对于 Kotlin，<!--
-->使用此方法使得无论挂起 lambda 表达式体内有多少挂起点，<!--
-->编译器也只创建一个类。

主要思想：挂起函数编译为状态机，其状态对应着挂起点。<!--
-->示例：编写一个有两个挂起点的挂起代码块：

```kotlin
val a = a()
val y = foo(a).await() // 挂起点 #1
b()
val z = bar(a, y).await() // 挂起点 #2
c(z)
```

这个代码块有三个状态：

* 初始化（在所有挂起点之前）
* 在第一个挂起点之后
* 在第二个挂起点之后

每个状态都是这个代码块某个续体的入口点<!--
-->（初始续体从第一行开始）。

代码会被编译为一个匿名类，它的一个方法实现了这个状态机、<!--
-->一个字段持有状态机当前状态，<!--
-->状态之间共享协程的局部变量字段（也可能有协程闭包的字段，<!--
--><u>但在这种情况下它是空的</u>*译者注：未能 get 到 `它` 指代什么*）。<!--
-->这是上文代码块通过续体传递风格调用挂起函数 `await` 的 Java 伪代码：

```java
lass <anonymous_for_state_machine> extends SuspendLambda<...> {
    // 状态机当前状态
    int label = 0
    
    // 协程的局部变量
    A a = null
    Y y = null
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        // 这次调用，result 应该为空
        a = a()
        label = 1
        result = foo(a).await(this) // 'this' 作为续体传递
        if (result == COROUTINE_SUSPENDED) return // 如果 await 挂起了执行则返回
      L1:
        // 外部代码传入 .await() 的结果恢复协程 
        y = (Y) result
        b()
        label = 2
        result = bar(a, y).await(this) // 'this' 作为续体传递
        if (result == COROUTINE_SUSPENDED) return // 如果 await 挂起了执行则返回
      L2:
        // 外部代码传入 .await() 的结果恢复协程
        Z z = (Z) result
        c(z)
        label = -1 // 没有其他步骤了
        return
    }          
}    
```

请注意，这里有 `goto` 操作符，还有标签，因为该示例描述的变化发生在<!--
-->字节码中而不是源码中。

现在，当协程开始时，我们调用了它的 `resumeWith()` —— `label` 是 `0`，<!--
-->然后我们跳去 `L0`，接着我们做一些工作，将 `label` 设为下一个状态—— `1`，调用 `.await()`，<!--
-->如果协程执行挂起就返回。<!--
-->当我们想继续执行时，我们再次调用 `resumeWith()`，现在它继续执行到了
`L1`，做一些工作，将状态设为 `2`，调用 `.await()`，同样在挂起时返回。<!--
-->下一次它从 `L3` 继续，将状态设为 `-1`，这意味着<!--
-->"结束了，没有更多工作要做了"。

循环内的挂起点只生成一个状态，<!--
-->因为循环（可能）也基于 `goto` 工作：

```kotlin
var x = 0
while (x < 10) {
    x += nextNumber().await()
}
```

生成为

```java
class <anonymous_for_state_machine> extends SuspendLambda<...> {
    // 状态机当前状态
    int label = 0
    
    // 协程局部变量
    int x
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        
      L0:
        x = 0
      LOOP:
        if (x > 10) goto END
        label = 1
        result = nextNumber().await(this) // 'this' 作为续体传递 
        if (result == COROUTINE_SUSPENDED) return // 如果 await 挂起了执行则返回
      L1:
        // 外部代码传入 .await() 的结果恢复协程
        x += ((Integer) result).intValue()
        label = -1
        goto LOOP
      END:
        label = -1 // 没有其他步骤了
        return 
    }          
}    
```

### 编译挂起函数

挂起函数代码在编译后的样子取决于它调用其他挂起函数的方式和时间。<!--
-->最简单的情况是一个挂起函数只在其*末尾*调用其他挂起函数，<!--
-->这称作对它们的*尾调用*。对于那些实现底层同步原语或者<!--
-->包装回调函数的协程来说，这是典型的方式，就像[挂起函数](#挂起函数)小节<!--
-->和[包装回调](#包装回调)小节展示的那样。这些函数在末尾<!--
-->像调用 `suspendCoroutine` 那样调用其他挂起函数。编译这种挂起函数就和编译普通的非挂起函数一样，<!--
-->唯一的区别是通过 [CPS 转换](#续体传递风格)拿到的隐式续体参数<!--
-->会在尾调用中传递给下一个挂起函数。

如果挂起调用出现的位置不是末尾，编译器将为挂起函数生成一个<!--
-->[状态机](#状态机)。状态机的实例<!--
-->在挂起函数调用时创建，在挂起函数完结时丢弃。

> 注意：在未来的版本中编译策略可能会优化成<!--
  -->在第一个挂起点生成状态机实例。

反过来，不在尾部调用其他挂起函数时，这个状态机又充当了*完结续体*。<!--
-->挂起函数多次调用其他挂起函数时，<!--
-->状态机实例会被更新并重用。<!--
-->对比其他[异步编程风格](#异步编程风格)，<!--
-->（其他异步编程风格中）异步过程的每个后续步骤通常使用单独的、新分配的<!--
-->闭包来实现。

### 协程内建函数

Kotlin 标准库提供了 `kotlin.coroutines.intrinsics` 包，其中包含许多<!--
-->声明，但应当谨慎使用，因为这些声明暴露了协程机制的内部实现细节。<!--
-->本节将解释这些细节。这些声明不应在通常的代码中使用，所以
`kotlin.coroutines.intrinsics` 包在 IDE 的自动补全中是被隐藏的。要使用<!--
-->这些声明，你必须手动把对应的 import 语句添加到源码文件：

 ```kotlin
 import kotlin.coroutines.intrinsics.*
 ```
 
标准库中的 `suspendCoroutine` 挂起函数的实际实现使用了 Kotlin 本身来编写，<!--
-->其源代码作为标准库源码包的一部分是可见的。为了<!--
-->安全、无问题地使用协程，它在协程每次挂起时将状态机的实际续体<!--
-->包装在一个附加对象中。这对于<!--
-->[异步计算](#异步计算)和 [Future](#Future) 等真正的异步用例来说非常好，因为<!--
-->相应异步原语的运行时开销远超分配一个额外的对象的开销。然而，对于<!--
-->[生成器](#生成器)用例，这个额外的消耗过高，因此内建函数包为<!--
-->性能敏感的底层代码提供了原语。

标准库 `kotlin.coroutines.intrinsics` 包中名为 <!--
-->[`suspendCoroutineUninterceptedOrReturn`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/suspend-coroutine-unintercepted-or-return.html) <!--
-->的函数拥有以下签名：

```kotlin
suspend fun <T> suspendCoroutineUninterceptedOrReturn(block: (Continuation<T>) -> Any?): T
```

它提供了对挂起函数的[续体传递风格](#续体传递风格)的直接访问，<!--
-->并且暴露了对*未拦截*的续体的引用。后者意味着 `Continuation.resumeWith` 的调用<!--
-->可以不通过[续体拦截器](#续体拦截器)。它可以用于<!--
-->编写[受限挂起](#受限挂起)的同步协程，因为这种协程不能安装<!--
-->续体拦截器（这又是因为它们的上下文始终为空），或者<!--
-->用在能确定当前执行线程就在所需的上下文中时（因为这时候也没必要拦截）。<!--
-->否则，应使用
[`intercepted`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/intercepted.html) <!--
-->扩展函数（位于 `kotlin.coroutines.intrinsics` 包）获取被拦截的续体：

```kotlin
fun <T> Continuation<T>.intercepted(): Continuation<T>
```

并且还应该在被*拦截*到的续体上调用 `Continuation.resumeWith`。

这时，如果协程确实挂起了，传递给 `suspendCoroutineUninterceptedOrReturn` 函数的 `block` 将返回
[`COROUTINE_SUSPENDED`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/-c-o-r-o-u-t-i-n-e_-s-u-s-p-e-n-d-e-d.html)<!--
-->（这种情况下，稍后对 `Continuation.resumeWith` 的调用应该有且仅有一次），否则，<!--
-->返回结果的值 `T` 或抛出一个异常（无论值还是异常，都不能再调用 `Continuation.resumeWith` 了）。

当使用 `suspendCoroutineUninterceptedOrReturn` 时，如果不遵守这一惯例，将导致难以跟踪的错误，<!--
-->而且与通过测试找到并复现错误的努力背道而驰。<!--
-->对于类似 `buildSequence`/`yield` 的协程来说，这种约定通常很容易遵循，<!--
-->但是**不建议**基于 `suspendCoroutineUninterceptedOrReturn` 编写类似异步 `await` 的挂起函数，因为如果没有
`suspendCoroutine` 的帮助，正确实现它们是**极难**的。

另有一些名为 <!--
-->[`createCoroutineUnintercepted`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/create-coroutine-unintercepted.html) <!--
-->（位于 `kotlin.coroutines.intrinsics` 包）<!--
-->的函数拥有以下签名：

```kotlin
fun <T> (suspend () -> T).createCoroutineUnintercepted(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(receiver: R, completion: Continuation<T>): Continuation<Unit>
```
 
它们的工作方式类似于 `createCoroutine` 但会返回对未拦截的初始续体的引用。<!--
-->类似于 `suspendCoroutineUninterceptedOrReturn`，它可用于同步协程以获得更好的性能。<!--
-->例如，下面是 `sequence{}` 构建器使用 `createCoroutineUnintercepted` 优化过的版本：
 
```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutineUnintercepted(receiver = this, completion = this)
    }
}
```

下面是 `yield` 使用 `suspendCoroutineUninterceptedOrReturn` 优化过的版本。<!--
-->注意，因为 `yield` 必定要挂起，对应的代码块也必定返回 `COROUTINE_SUSPENDED`。

```kotlin
// 实现生成器
override suspend fun yield(value: T) {
    setNext(value)
    return suspendCoroutineUninterceptedOrReturn { cont ->
        nextStep = cont
        COROUTINE_SUSPENDED
    }
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/optimized/sequenceOptimized.kt)获取完整代码

另外两个内建函数提供 `startCoroutine`（查看[协程构建器](#协程构建器)一节）的底层版本，<!--
-->名为：<!--
-->[`startCoroutineUninterceptedOrReturn`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/start-coroutine-unintercepted-or-return.html)：

```kotlin
fun <T> (suspend () -> T).startCoroutineUninterceptedOrReturn(completion: Continuation<T>): Any?
fun <R, T> (suspend R.() -> T).startCoroutineUninterceptedOrReturn(receiver: R, completion: Continuation<T>): Any?
```

它们在两方面不同于 `startCoroutine`。首先，[续体拦截器](#续体拦截器)<!--
-->在开启协程时不会自动使用，因此如果需要，调用方必须确保执行上下文的正确性。<!--
-->其次，如果协程没有挂起，而是返回一个值或抛出异常，那么<!--
-->调用 `startCoroutineUninterceptedOrReturn` 会返回这个值或抛出这个异常。如果协程<!--
-->挂起了，将会返回 `COROUTINE_SUSPENDED`。

`startCoroutineUninterceptedOrReturn` 的基本用例是与 `suspendCoroutineUninterceptedOrReturn` 结合，<!--
-->在具有相同上下文的不同代码块中继续运行挂起的协程：

```kotlin 
suspend fun doSomething() = suspendCoroutineUninterceptedOrReturn { cont ->
    // 找到或创建需要运行的代码块
    startCoroutineUninterceptedOrReturn(completion = block) // 将结果返回到 suspendCoroutineUninterceptedOrReturn
}
```

## 附录

这是非规范性的部分，不引入新的语言结构或<!--
-->库函数，而是讨论了一些涉及资源管理、并发<!--
-->和编码风格的话题，并为各种各样的用例提供了更多示例。

### 资源管理与垃圾收集

协程不使用堆外存储，也不自行消耗任何本地资源，除非<!--
-->在协程中运行的代码打开了文件或占用了其他资源。在协程中打开的文件必然<!--
-->要以某种方式关闭，这不意味着协程本身需要关闭。当协程挂起时，其状态<!--
-->可以通过对其续体的引用来获取。如果你失去了对挂起协程续体的引用，<!--
-->最终它会被垃圾收集器回收。

打开了可关闭资源的协程应该特别关注。考虑下面这个<!--
-->[受限挂起](#受限挂起)小节中使用 `sequence{}` 构建器从文件生成<!--
-->行序列的协程：

```kotlin
fun sequenceOfLines(fileName: String) = sequence<String> {
    BufferedReader(FileReader(fileName)).use {
        while (true) {
            yield(it.readLine() ?: break)
        }
    }
}
```

这个函数返回一个 `Sequence<String>`，通过这个函数，你可以<!--
-->用一种自然的方式打印文件的所有行：

```kotlin
sequenceOfLines("https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt")
    .forEach(::println)
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt)获取完整代码

只要你遍历 `sequenceOfLines` 函数返回的整个序列，它就工作正常。<!--
-->然而，如果你只打印了文件的前几行，就像这样：

```kotlin
sequenceOfLines("https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt")
        .take(3)
        .forEach(::println)
```

协程恢复了几次，产生出文件的前三行，然后就被*遗弃*了。<!--
-->遗弃对于协程本身来说没什么关系，但是对于打开了的文件则不然。<!--
-->[`use` 函数](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html)<!--
-->没有机会结束调用并关闭文件。文件会一直处于开启状态，<!--
-->直到被垃圾收集器回收，因为 Java 文件有个能关闭文件的 `finalizer`。<!--
-->如果只是个幻灯片或者短时间运行的小工具，这倒也不是什么问题，但是对于那些有数 GB 堆容量的大型后端系统来说，<!--
-->可就是个灾难了，<!--
-->它可能会快速耗尽打开文件句柄会而不是耗尽内存触发垃圾收集。

这个问题和 Java
中生成行的惰性流的 [`Files.lines`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#lines-java.nio.file.Path-)
方法遇到的问题一样。它返回一个可关闭的 Java 流，但多数流操作不会<!--
-->自动调用对应的
`stream.close` 方法，需要用户自己记着关闭流。<!--
-->Kotlin 中也可以定义需要关闭的序列生成器，<!--
-->但也会遇到同一个问题，就是语言没有什么自动机制能<!--
-->保证它们在用完之后关闭。引入一种自动化资源管理的语言机制<!--
-->明显超出了 Kotlin 协程的领域。

然而，通常这个问题不会影响协程的异步用例。异步协程是<!--
-->不会被遗弃的，它会持续运行直到完毕。因此只要协程中的代码能正确地关闭<!--
-->其资源，资源最终就会被关闭。

### 并发和线程

在一个独立协程的内部，如同线程内部一样，是顺序执行的。这意味着下面这种<!--
-->协程内的代码是相当安全的：

```kotlin
launch { // 启动协程
    val m = mutableMapOf<String, String>()
    val v1 = someAsyncTask1() // 开始一些异步任务
    val v2 = someAsyncTask2() // 开始一些异步任务
    m["k1"] = v1.await() // 修改映射等待操作完成
    m["k2"] = v2.await() // 修改映射等待操作完成
}
```

在协程的作用域里，你可以随意使用那些普通的单线程的可变结构。<!--
-->然而，在协程*之间*共享可变状态仍可能带来致命威胁。如果你使用了<!--
-->一个指定调度器的协程构建器，以 JS 风格在单一事件调度线程上恢复协程，<!--
-->就像[续体拦截器](#续体拦截器)一节展示的 `Swing` 拦截器那样，<!--
-->那你还是能安全地操作所有共享对象，<!--
-->因为它们总在事件调度线程上修改。<!--
-->但如果你在多线程环境中，或者需要<!--
-->运行在不同线程上的多个协程之间共享可变状态，你就必须使用线程安全（并发）的数据结构。

协程在这方面和线程没有什么不同，尽管协程确实更轻。你可以在仅仅几个线程上<!--
-->同时运行几百万个协程。一个运行着的协程总是在某个线程上。但一个*挂起*了的协程<!--
-->并不占用线程，也没有以任何方式绑定到线程。恢复协程的挂起函数<!--
-->通过在线程上调用 `Continuation.resumeWith` 决定在哪个线程上恢复协程。<!--
-->而协程的拦截器可以覆盖这个决定，并将协程的执行调度到另外的线程上。

### 异步编程风格

异步编程有多种不同的风格。

[异步计算](#异步计算)一节已经讨论了回调函数，这通常也是协程被设计出来<!--
-->替换的最不方便的一种风格。任何回调风格的应用程序接口都可以<!--
-->用对应的挂起函数包装，见[这里](#包装回调)。

我们来回顾一下。假设你现在有一个带有以下签名的*阻塞* `sendMail` 函数：


```kotlin
fun sendEmail(emailArgs: EmailArgs): EmailResult
```

它在运行时可能会阻塞执行线程很长的时间。

要使其不阻塞，可以使用错误优先的
[node.js 回调约定](https://www.tutorialspoint.com/nodejs/nodejs_callbacks_concept.htm)，<!--
-->以回调风格表示其非阻塞版本，签名如下：

```kotlin
fun sendEmail(emailArgs: EmailArgs, callback: (Throwable?, EmailResult?) -> Unit)
```

然而，协程还能支持其他风格的异步非阻塞编程。其中之一<!--
-->是内置于许多广受欢迎的语言中的 async/await 风格。<!--
-->在 Kotlin 中，可以通过引入 `future{}` 和 `.await()` 库函数<!--
-->来重现这种风格，就像用例中的 [future](#Future) 小节所示。

这种风格主张从函数返回对未来对象的某种约定，<!--
-->而不是传入回调函数作为参数。在这种异步风格中，`sendEmail` 的签名看起来是这样：

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult>
```

作为一种风格，将 Async 后缀添加到此类方法名称是一个好习惯，因为它们的<!--
-->参数和阻塞版本没什么不同，因而很容易犯忘记其本质是<!--
-->异步操作的错误。函数 `sendEmailAsync` 启动一个*并发*异步的操作，<!--
-->可能带来并发的所有陷阱。然而，鼓励这种风格的编程语言<!--
-->通常也提供某种 `await` 原语，在需要的时候把操作重新变回顺序的。

Kotlin 的*原生*编程风格基于挂起函数。在这种风格下，`sendEmail` 的签名<!--
-->看起来比较自然，不修改其参数或返回类型，而是增加了一个
`suspend` 修饰符：

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult
```

我们已经发现，async 和挂起风格可以通过原语很容易地相互转换。<!--
-->例如，挂起版本的 `sendEmail` 可以用 [future 构建器](#构建-Future)轻松实现 `sendEmailAsync`：


```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult> = future {
    sendEmail(emailArgs)
}
```

`sendEmailAsync` 使用 [`.await()` 挂起函数](#挂起函数)也能实现挂起函数 `sendEmail`：


```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult = 
    sendEmailAsync(emailArgs).await()
```

因此，在某种意义上，这两种风格是等效的，并且在方便性上都明显优于回调风格。<!--
-->然而，我们还可以更深入地研究 `sendEmailAsync` 和挂起的 `sendEmail` 之间的区别。

让我们先比较一下他们在代码中**使用**的方式。挂起函数可以像普通函数一样使用：

```kotlin
suspend fun largerBusinessProcess() {
    // 这里有很多代码，接下来在某处……
    sendEmail(emailArgs)
    // ……后来又继续做了些别的事
}
```

对应的异步风格函数这样使用：

```kotlin
fun largerBusinessProcessAsync() = future {
    // 这里有很多代码，接下来在某处……
    sendEmailAsync(emailArgs).await()
    // ……后来又继续做了些别的事
}
```

显然，异步风格的函数结构更冗长，更容易出错。<!--
-->如果在异步风格的示例中省略了 `.await()` 调用，<!--
-->代码仍然可以编译并工作，但现在它通过发送电子邮件来处理异步，<!--
-->*同时*甚至在执行其余更大的业务流程，<!--
-->因此可能会修改某些共享状态并引入一些非常难以重现的错误。<!--
-->相反，挂起函数是*默认顺序*的。<!--
-->对于挂起的函数，无论何时需要任何并发，都可以在代码中通过调用<!--
-->某种 `future{}` 或类似的协程构建器显式地表达。

从使用多个库的大型项目的**规模**来比较这些风格。挂起函数是 <!--
-->Kotlin 的一个轻量级语言概念。所有挂起函数在任何非限定性的 Kotlin 协程中都是完全可用的。<!--
-->async 风格的函数依赖于框架。每个 promises/futures 框架都必须定义自己的——类`async` 函数，<!--
-->该函数返回自己的 promise/future 类，这些类又有对应的——类 `async` 函数。

从**性能**比较。挂起函数拥有最小的调用开销。<!--
-->你可以查看[实现细节](#实现细节)小节。<!--
-->除了必要的挂起机制之外，async 风格的函数需要额外维护相当重的 promise/future 抽象。<!--
-->async 风格的函数调用必须返回一些类似 future 的实例对象，并且即使函数非常简短，<!--
-->也无法将其优化。异步风格不太适合于粒度非常细的分解。

从与 JVM/JS 代码的**互操作性**比较。async 风格的函数与 JVM/JS 代码更具互操作性，<!--
-->因为这类代码的类型系统匹配 future 的抽象。在 Java 或 JS 中，它们只是返回<!--
-->类似 future 的对象的函数。对任何不原生支持<!--
-->[续体传递风格](#续体传递风格)的语言来说，挂起函数都很奇怪。<!--
-->然而，从上面的示例中可以看出，对于任何给定的 promise/future 框架都很容易将任何挂起函数转换为
async 风格的函数。因此，只要用 Kotlin 编写一次挂起函数，<!--
-->然后使用适当的 `future{}` 协程构建器函数通过一行代码对其进行调整，<!--
-->就能实现与任何形式的 promise/future 的互操作性。

### 包装回调

很多异步应用程序接口集包含回调风格的接口。标准库中的<!--
-->挂起函数 `suspendCoroutine`（见[挂起函数](#挂起函数)小节）<!--
-->提供了一种简单的把任何回调函数包装成 Kotlin 挂起函数的方法。

这里有一个简单的例子。有一个简单的模式。假设你有一个带有回调的 `someLongComputation` 函数，<!--
-->回调接收一些作为计算的结果的 `Value`。

```kotlin
fun someLongComputation(params: Params, callback: (Value) -> Unit)
```

你可以用下面这样的代码直截了当的把它变成挂起函数：

```kotlin
suspend fun someLongComputation(params: Params): Value = suspendCoroutine { cont ->
    someLongComputation(params) { cont.resume(it) }
} 
```

现在计算的返回值变成显式的了，但它仍然是异步的，且不会阻塞线程。

> 注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 包含了一个<!--
  -->协作式可取消协程框架。它提供类似 `suspendCoroutine`，<!--
  -->但支持取消的 `suspendCancellableCoroutine` 函数。查看<!--
  -->其指南中[取消与超时](https://www.kotlincn.net/docs/reference/coroutines/cancellation-and-timeouts.html)一文<!--
  -->了解更多细节。

举一个更复杂的例子，我们看看<!--
-->[异步计算](#异步计算)用例中的 `aRead()` 函数。<!--
-->它可以实现为 Java NIO 中
[`AsynchronousFileChannel`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/AsynchronousFileChannel.html) <!--
-->的挂起扩展函数，它的
[`CompletionHandler`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/CompletionHandler.html)
回调接口如下：

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

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/io/io.kt)获取代码。<!--
  -->注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)
  中实际的实现支持取消以放弃长时间运行的 IO 操作。

如果你需要处理大量有同类回调的函数，你可以定义一个公共<!--
-->包装函数简便地把他们全部转换成挂起函数。例如，<!--
-->[vert.x](http://vertx.io/) 有一个特有的约定，其中所有异步函数都接收一个<!--
--> `Handler<AsyncResult<T>>` 回调。要通过协程简化任意的 vert.x 函数，<!--
-->可以定义下面这个辅助函数：

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

使用这个辅助函数，任意异步 vert.x 函数 `async.foo(params, handler)`
都可以在协程中这样调用：`vx { async.foo(params, it) }`。

### 构建 Future

定义在 [future](#Future) 用例中类似于 `launch{}` 构建器的 `future{}` 构建器可以用于实现任何 future 或 promise 原语，<!--
-->这在[协程构建器](#协程构建器)中做了一些介绍：

```kotlin
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
        CompletableFutureCoroutine<T>(context).also { block.startCoroutine(completion = it) }
```

它与 `launch{}` 的第一点不同是它返回
[`CompletableFuture`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 的实例，<!--
-->第二点不同是它包含一个默认为 `CommonPool` 的上下文，因此<!--
-->其默认执行在 [`ForkJoinPool.commonPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--)，<!--
-->这个默认执行行为类似于
[`CompletableFuture.supplyAsync`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) 方法。<!--
--><!--
-->`CompletableFutureCoroutine` 的基本实现很直白：

```kotlin
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
    override fun resumeWith(result: Result<T>) {
        result
            .onSuccess { complete(it) }
            .onFailure { completeExceptionally(it) }
    }
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/future/future.kt)获取代码。<!--
  -->[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中实际的实现更高级，<!--
  -->因为它传播对执行结果的可取消的 future，以取消该协程。

协程完结时调用对应 future 的 `complete` 方法<!--
-->向协程报告结果。

### 非阻塞休眠

协程不应使用 [`Thread.sleep`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-)，<!--
-->因为它阻塞了线程。但是，通过 Java 的 [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)
实现挂起的非阻塞 `delay` 函数是非常直截了当的。

```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
    Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont ->
    executor.schedule({ cont.resume(Unit) }, time, unit)
}
```

> 你可以从[这里](https://github.com/Kotlin/kotlin-coroutines/blob/master/examples/delay/delay.kt)获取到这段代码。<!--
  -->注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 同样提供了 `delay` 函数。

注意，这种 `delay` 函数从其单独的“调度者”线程恢复协程。<!--
-->那些使用[拦截器](#续体拦截器)的协程，比如 `Swing`，不会在这个线程上执行，<!--
-->因为它们的拦截器在合适的线程上调度它们。没有拦截器的协程会<!--
-->在调度者线程上调度。所以这对一个示例来说挺方便的，但它不是最有效的。<!--
-->最好能在相应的拦截器中实现原生的休眠。

对于 `Swing` 拦截器，非阻塞休眠的原生实现应使用专门为此目的设计的 <!--
-->[`Swing 计时器`](https://docs.oracle.com/javase/8/docs/api/javax/swing/Timer.html)：


```kotlin
suspend fun Swing.delay(millis: Int): Unit = suspendCoroutine { cont ->
    Timer(millis) { cont.resume(Unit) }.apply {
        isRepeats = false
        start()
    }
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/swing-delay.kt)获取到这段代码。<!--
  -->注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的 `delay` 实现注意了<!--
  -->拦截器特异性的休眠机制，并在适当的情况下自动使用上述方法。

### 协作式单线程多任务

在单线程应用中实现多任务非常方便，因为这样就不必<!--
-->处理并发或者共享可变状态了。JS、Python 以及很多其他语言<!--
-->没有线程，但有协作式多任务原语。

[协程拦截器](#续体拦截器)提供了一个简单的工具来确保<!--
-->所有协程被限制在一个单线程上。<!--
-->[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/threadContext.kt)的示例代码定义了 `newSingleThreadContext()` 函数，<!--
-->它能创建一个单线程执行的服务并使其适应协程拦截器的<!--
-->需求。

在下面的示例中，<!--
-->我们把它和[构造 Future](#构造-Future)小节中定义的 `future{}` 协程构建器一起使用，使其运行在一个单个线程中<!--
-->尽管它有两个同时处于活动状态的异步任务。

```kotlin
fun main(args: Array<String>) {
    log("Starting MyEventThread")
    val context = newSingleThreadContext("MyEventThread")
    val f = future(context) {
        log("Hello, world!")
        val f1 = future(context) {
            log("f1 is sleeping")
            delay(1000) // 休眠 1 秒
            log("f1 returns 1")
            1
        }
        val f2 = future(context) {
            log("f2 is sleeping")
            delay(1000) // 休眠 1 秒
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

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/threadContext-example.kt)获取完整示例。<!--
  -->注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 有
  `newSingleThreadContext` 开箱即用的实现。

如果你的整个应用都在同一个线程上执行，你可以定义自己的辅助协程构建器，<!--
-->在其中硬编码一个适应你单线程执行机制的上下文。

### 异步序列

[受限挂起](#受限挂起)小节展示的 `sequence{}` 协程构建器<!--
-->是一个*同步*协程的示例。当消费者调用 `Iterator.next()` 时，<!--
-->协程的生产代码同步执行在同一个线程上。<!--
-->`sequence{}` 协程块是受限的，第三方挂起<!--
-->函数无法挂起其执行，比如[包装回调](#包装回调)一节中那种异步文件 IO。

*异步的*序列构建器允许随意挂起和恢复执行。这意味着<!--
-->其消费者要时刻准备着处理数据还没生产出来的情况。这是<!--
-->挂起函数的自然用例。我们来定义一个<!--
-->类似于普通
[`Iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterator/)
接口的 `SuspendingIterator` 接口，但其 `next()` 和 `hasNext()` 函数是挂起的：

```kotlin
interface SuspendingIterator<out T> {
    suspend operator fun hasNext(): Boolean
    suspend operator fun next(): T
}
```

`SuspendingSequence` 的定义类似于标准
[`Sequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence/index.html)
但它返回 `SuspendingIterator`：

```kotlin
interface SuspendingSequence<out T> {
    operator fun iterator(): SuspendingIterator<T>
}
```

就像同步序列的作用域一样，我们也给它定义一个作用域接口，<!--
-->但它的挂起不是受限的：

```kotlin
interface SuspendingSequenceScope<in T> {
    suspend fun yield(value: T)
}
```

构建器函数 `suspendingSequence{}` 的用法和同步的 `sequence{}` 一样。<!--
-->它们的区别在于 `SuspendingIteratorCoroutine` 的实现细节以及<!--
-->在下面这种情况中，以及在这种情况下接受一个可选的上下文是有意义的：

```kotlin
fun <T> suspendingSequence(
    context: CoroutineContext = EmptyCoroutineContext,
    block: suspend SuspendingSequenceScope<T>.() -> Unit
): SuspendingSequence<T> = object : SuspendingSequence<T> {
    override fun iterator(): SuspendingIterator<T> = suspendingIterator(context, block)
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/suspendingSequence/suspendingSequence.kt)获取完整代码。<!--
  -->注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中对
  `Channel` 原语的实现使用了对应的协程构建器 `produce{}`，<!--
  -->其中对这个概念提供了更复杂的实现。

我们可以用[单线程多任务](#单线程多任务)小节中的 `newSingleThreadContext{}` 上下文<!--
-->和[非阻塞睡眠](#非阻塞睡眠)小节的非阻塞的 `delay` 函数。<!--
--><!--
-->这样我们就能编写一个非阻塞序列的实现来生产
1 ~ 10 的整数，两数之间间隔 500 毫秒：

```kotlin
val seq = suspendingSequence(context) {
    for (i in 1..10) {
        yield(i)
        delay(500L)
    }
}
```

现在消费者协程可以按自己喜欢的方式消费序列了，也可以<!--
-->被任意的挂起函数挂起。注意，<!--
-->Kotlin [for 循环](https://kotlinlang.org/docs/reference/control-flow.html#for-loops)<!--
-->的工作方式满足这种序列的约定，因此语言中不需要一个专门的 `await for` 循环结构。<!--
-->普通的 `for` 循环就能用来遍历我们在上面定义的异步序列。<!--
-->生产者没有值的时候它就会挂起：


```kotlin
for (value in seq) { // 等待生产者生产时挂起
    // 在这里用值做些事，也可以在这里挂起
}
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/suspendingSequence/suspendingSequence-example.kt)找到带有一些日志的示例，<!--
  -->说明此处的执行情况，。

### 通道

Go 风格的类型安全通道在 Kotlin 中通过库实现。我们可以为<!--
-->发送通道定义一个接口，包含挂起函数 `send`：

```kotlin
interface SendChannel<T> {
    suspend fun send(value: T)
    fun close()
}
```

以及风格类似[异步序列](#异步序列)的接收通道，<!--
-->包含挂起函数 `receive` 和 `operator iterator`：

```kotlin
interface ReceiveChannel<T> {
    suspend fun receive(): T
    suspend operator fun iterator(): ReceiveIterator<T>
}
```

`Channel<T>` 类同时实现这两个接口。<!--
-->通道缓存满时 `send` 挂起，通道缓存空时 `receive` 挂起。<!--
-->这样我们可以一字不差地复制 Go 风格的代码。<!--
-->[Go 教程的第 4 个并发示例](https://tour.golang.org/concurrency/4)<!--
-->中向通道发送 n 个斐波那契数的 `fibonacci` 函数用 Kotlin 实现<!--
-->看起来是这样：

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

我们也可以定义 Go 风格的 `go {...}` 代码块在某种线程池上启动新协程，<!--
-->在固定数量的重量线程上调度任意多的轻量协程。<!--
-->[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/go.kt)<!--
-->的示例实现简单地<!--
-->在 Java 通用的 [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html) 之上编写。

使用 `go` 协程构建器，对应的 Go 代码主函数看起来是下面这样，<!--
-->其中的`mainBlocking` 是简化的辅助函数，它在 `go{}` 的线程池上调用 `runBlocking`：

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val c = Channel<Int>(2)
    go { fibonacci(10, c) }
    for (i in c) {
        println(i)
    }
}
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-4.kt)查看代码

你可以随意修改通道的缓冲区容量。<!--
-->为了简化，例子中只实现了缓冲通道（最小缓存 1 个值），<!--
-->因为无缓冲通道在概念上和我们刚才见过的[异步序列](#异步序列)一样。


Go 风格的 `select` 控制流，作用是挂起直到其中一个操作在其中一个通道上可用，<!--
-->可以实现为 Kotlin DSL，<!--
-->因此 [Go 教程的第 5 个并发示例](https://tour.golang.org/concurrency/5)<!--
-->在 Kotlin 中看起来是这样：

```kotlin
suspend fun fibonacci(c: SendChannel<Int>, quit: ReceiveChannel<Int>) {
    var x = 0
    var y = 1
    whileSelect {
        c.onSend(x) {
            val next = x + y
            x = y
            y = next
            true // 继续 while 循环
        }
        quit.onReceive {
            println("quit")
            false // 退出 while 循环
        }
    }
}
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-5.kt)查看代码

示例具有 `select {...}` 的实现，它选择一种情况并返回结果，就像 Kotlin 的
[`when` 表达式](https://kotlinlang.org/docs/reference/control-flow.html#when-expression)，<!--
-->还用到了一个方便的 `whileSelect { ... }`，它就是 `while(select<Boolean> { ... })`，但需要的括号比较少。

实现 [Go 教程的第 6 个并发示例](https://tour.golang.org/concurrency/6)中的默认选项<!--
-->只需添加另一个选项到 `select {...}` DSL：

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val tick = Time.tick(100)
    val boom = Time.after(500)
    whileSelect {
        tick.onReceive {
            println("tick.")
            true // 继续循环
        }
        boom.onReceive {
            println("BOOM!")
            false // 继续循环
        }
        onDefault {
            println("    .")
            delay(50)
            true // 继续循环
        }
    }
}
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-6.kt)查看代码

[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/time.kt)<!--
-->的 `Time.tick` 和 `Time.after` 用非阻塞的 `delay` 函数实现非常简单。

[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/)能找到其他示例，<!--
-->注释里有对应的 Go 代码的链接。

注意，这是通道的简单实现，只用了一个<!--
-->锁来管理内部的等待队列。这使得它容易理解和解释。<!--
-->然而，它并不在这个锁下运行用户代码，因此它是完全并发的。<!--
-->这个锁只在一定程度上限制了它对大量并发线程的可伸缩性。

> 通道和 `select` 在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) <!--
-->中的实际实现基于无锁的无冲突并发访问数据结构。

这样实现的通道独立于<!--
-->协程上下文中的拦截器。它可以用于 UI 应用程序，<!--
-->通过[续体拦截器](#续体拦截器)小节<!--
-->提到的事件线程拦截器，或者任何别的拦截器，或者不使用<!--
-->任何拦截器也可以（在后一种情况下，实际的执行线程完全由<!--
-->协程中使用的其他挂起函数的代码决定）。<!--
-->通道实现提供的挂起函数都是非阻塞且线程安全的。

### 互斥

编写可伸缩的异步应用程序应遵循一个原则，确保<!--
-->代码挂起（使用挂起函数）而不阻塞，即实际上不阻塞线程。<!--
-->Java 并发原语
[`ReentrantLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)
阻塞线程，不应在真正的非阻塞代码中使用。要控制对共享<!--
-->资源的访问，可以定义一个 `Mutex` 类，该类挂起协程的执行，而不是阻塞协程。<!--
-->这个类的声明看起来是这样：

```kotlin
class Mutex {
    suspend fun lock()
    fun unlock()
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/mutex/mutex.kt)获得完整的实现。<!--
  -->在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实际实现<!--
  -->还包含一些其他的函数。

使用这个非阻塞互斥的实现，<!--
-->[Go 教程的第 9 个并发示例](https://tour.golang.org/concurrency/9)<!--
-->可以用 Kotlin 的 <!--
-->[`try finally`](https://kotlinlang.org/docs/reference/exceptions.html) <!--
-->翻译到 Kotlin，这与 Go 的 `defer` 作用相同：

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

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-9.kt)查看代码

### 从实验性协程移植

协程在 Kotlin 1.1-1.2 是一个实验特性。相关的应用程序接口<!--
-->位于 `kotlin.coroutines.experimental` 包。随 Kotlin 1.3 推出的稳定版本的协程<!--
-->位于 `kotlin.coroutines`。标准库中的实验性包仍然可用，并且<!--
-->用实验性协程编译的代码的行为也和以前一样。
 
Kotlin 1.3 编译器支持调用实验挂起函数，并将挂起 <!--
-->lambdas 表达式传递给用实验性协程编译的库。<!--
-->在幕后，我们创建了对应的稳定版与实验性协程接口之间的适配器。

### 参考

* 扩展阅读：
   * **先读这个**[协程指南](https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html)！。
* 介绍：
   * [初识协程](https://www.youtube.com/watch?v=_hfBv0a09Jc)（Roman Elizarov，于 KotlinConf 2017，[幻灯片](https://www.slideshare.net/elizarov/introduction-to-coroutines-kotlinconf-2017)）
   * [深入协程](https://www.youtube.com/watch?v=YrrUCSi72E8)（Roman Elizarov，于 KotlinConf 2017，[幻灯片](https://www.slideshare.net/elizarov/deep-dive-into-coroutines-on-jvm-kotlinconf-2017)）
   * [实践协程](https://www.youtube.com/watch?v=a3agLJQ6vt8)（Roman Elizarov，于 KotlinConf 2018，[幻灯片](https://www.slideshare.net/elizarov/kotlin-coroutines-in-practice-kotlinconf-2018)）
* 语言设计概述：
  * 第 1 部分（原型设计）：[Kotlin 中的协程](https://www.youtube.com/watch?v=4W3ruTWUhpw)
   （Andrey Breslav，于 JVMLS 2016）
  * 第 2 部分（当前设计）：[Kotlin 协程新生](https://www.youtube.com/watch?v=3xalVUY69Ok&feature=youtu.be)
   （Roman Elizarov，于 JVMLS 2017，[幻灯片](https://www.slideshare.net/elizarov/kotlin-coroutines-reloaded)）

### 反馈

请将反馈提交到：

* [Kotlin YouTrack](http://kotl.in/issue) 关于 Kotlin 编译器中协程的实现和特性的意见。
* [`kotlinx.coroutines`](https://github.com/Kotlin/kotlinx.coroutines/issues) 关于支持库的意见。

## 版本历史

本节概述了协程设计的诸次修订之间的变化。

### 3.3 版本的改动

> * Coroutines are no longer experimental and had moved to `kotlin.coroutines` package.
> * The whole section on experimental status is removed and migration section is added.
> * Some non-normative stylistic changes to reflect evolution of naming style.
> * Specifications are updated for new features implemented in Kotlin 1.3:
>   * More operators and different types of functions are supports.
>   * Changes in the list of intrinsic functions:
>   * `suspendCoroutineOrReturn` is removed, `suspendCoroutineUninterceptedOrReturn` is provided instead.
>   * `createCoroutineUnchecked` is removed, `createCoroutineUnintercepted` is provided instead.
>   * `startCoroutineUninterceptedOrReturn` is provided.
>   * `intercepted` extension function is added.
> * Moved non-normative sections with advanced topics and more examples to the appendix at end of the document to simplify reading.

### 3.2 版本的改动

> * Added description of `createCoroutineUnchecked` intrinsic.

### 3.1 版本的改动

This revision is implemented in Kotlin 1.1.0 release.

> * `kotlin.coroutines` package is replaced with `kotlin.coroutines.experimental`.
> * `SUSPENDED_MARKER` is renamed to `COROUTINE_SUSPENDED`.
> * Clarification on experimental status of coroutines added.

### 3 版本的改动

> This revision is implemented in Kotlin 1.1-Beta.
> 
> * Suspending functions can invoke other suspending function at arbitrary points.
> * Coroutine dispatchers are generalized to coroutine contexts:
>   * `CoroutineContext` interface is introduced.
>   * `ContinuationDispatcher` interface is replaced with `ContinuationInterceptor`.
>   * `createCoroutine`/`startCoroutine` parameter `dispatcher` is removed.
>   * `Continuation` interface includes `val context: CoroutineContext`.
> * `CoroutineIntrinsics` object is replaced with `kotlin.coroutines.intrinsics` package.

### 2 版本的改动

> This revision is implemented in Kotlin 1.1-M04.
> 
> * The `coroutine` keyword is replaced by suspending functional type.
> * `Continuation` for suspending functions is implicit both on call site and on declaration site.
> * `suspendContinuation` is provided to capture continuation is suspending functions when needed.
> * Continuation passing style transformation has provision to prevent stack growth on non-suspending invocations.
> * `createCoroutine`/`startCoroutine` coroutine builders are introduced.
> * The concept of coroutine controller is dropped:
>   * Coroutine completion result is delivered via `Continuation` interface.
>   * Coroutine scope is optionally available via coroutine `receiver`.
>   * Suspending functions can be defined at top-level without receiver.
> * `CoroutineIntrinsics` object contains low-level primitives for cases where performance is more important than safety.

## 名词对照

| 原文                       | 译文          | 语境                      |
| :------------------------: | :----------: | :-----------------------: |
| complete / completion      | 完结         | 主语是协程，或协程包装的东西 |
| continuation               | 续体         | 所有                       |
| continuation passing style | 续体传递风格  | 所有                       |
| coroutine intrinsics       | 协程内建函数  | 所有                       |
