# Kotlin 协程

* **类型**：设计提案
* **作者**：Andrey Breslav, Roman Elizarov
* **贡献者**： Vladimir Reshetnikov, Stanislav Erokhin, Ilya Ryzhenkov, Denis Zharkov
* **状态**：从 Kotlin 1.3 开始稳定，在 Kotlin 1.1-1.2 中为实验性

## 摘要

本文描述 Kotlin 协程。这一概念通常被认为与下列内容有关，或部分涵盖它们：

- generators/yield
- async/await
- composable/delimited сontinuations

设计目标：

- 不依赖 Future 之类复杂的库提供的特定设施；
- 同时涵盖 “async/await” 用例和 “生产者代码块”；
- 使 Kotlin 协程能包装各种现有的异步 API 
 （如 Java NIO、各种 Future 的实现等）；

## 目录

* [用例](#用例)
  * [异步计算](#异步计算)
  * [Future](#Future)
  * [生产者](#生产者)
  * [异步用户界面](#异步用户界面)
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
  * [非阻塞睡眠](#非阻塞睡眠)
  * [协作式单线程多任务](#协作式单线程多任务)
  * [异步序列](#异步序列)
  * [通道](#通道)
  * [互斥](#互斥)
  * [从实验性协程移植](#从实验性协程移植)
  * [参考](#参考)
  * [反馈](#反馈)
* [版本历史](#版本历史)
  * [3.3 版中的修改](#3.3-版中的修改)
  * [3.2 版中的修改](#3.2-版中的修改)
  * [3.1 版中的修改](#3.1-版中的修改)
  * [3 版中的修改](#3-版中的修改)
  * [2 版中的修改](#2-版中的修改)

## 用例

协程可以被视作*可挂起的计算* 的实例。换句话说，可以在某些<!--
-->点上挂起，稍后在另一个线程上恢复执行。协程相互调用<!--
-->（来回传递数据），即可形成<!--
-->多任务协同机制。

### 异步计算

最能描述协程功能的用例是异步计算<!--
-->（在 C# 及其他语言中通过 `async`/`await` 实现）。<!--
-->让我们来看看这种计算是如何通过回调来完成的。不妨以<!--
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
-->必要将  `buf ` 参数显式传递到回调，它们被看作是闭包的一部分），但缩进<!--
-->级别每次都在增长，而且只要嵌套超<!--
-->过一层，大家都知道能产生多少麻烦（百度“回调地狱”，看看 JavaScript 迫害了多少人）。

同样的计算可以直截了当地表达为协程（前提是有一个合适的库，<!--
-->使输入输出应用程序接口适配协程的需求）：

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

这里的 `aRead()` 和 `aWrite()` 是特殊的*挂起函数* —— 它们可以*挂起* 代码执行<!--
-->（这并不意味着阻塞正在运行它的线程），然后在调用完成时*恢复* 代码执行。<!--
-->如果我们眯起眼睛，可以想象所有在 `aRead()` 之后的代码已经被包装成一个 <!--
-->lambda 表达式并作为回调传递给 `aRead()`，对 `aWrite()` 也是如此，<!--
-->我们就可以看到这个代码和上面的一样，可读性却更强。

我们的明确目标是以一种非常通用的方式支持协程，所以在这个例子中，<!--
-->`launch{}`、`.aRead()` 和 `.aWrite()` 只是适应协程工作的**库函数**；<!--
-->`launch` 是*协程构建器* —— 它创建并启动协程，<!--
-->而 `aRead()` 和 `aWrite()` 作为特殊的<!--
-->*挂起函数* 隐式地接受<!--
-->*续体*（[续体](https://aisia.moe/2018/02/08/kotlin-coroutine-kepa/)就是一般的回调）。

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

还有另一种表达异步计算的方式：通过 future（或者叫做 promise 或 deferred）。<!--
-->我们将在在示例中使用一个虚构的应用程序接口，对图像应用一个覆盖：

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

再一次，协程对 future 的支持减少了缩进级别、逻辑（以及异常处理，这里没有出现）更加自然，<!--
-->而且没有使用专门的关键字（比如 C#、JS 以及其他语言中的 `async` 和 `await`）来支持 future：<!--
-->`future{}` 和 `.await()` 都只是库函数而已。

### 生产者

协程的另一个典型用例是延时计算序列（在 C#、Python <!--
-->和很多其他语言中通过 `yield`  实现）。这样的序列可以由看似连续的代码生成，但在运行时只<!--
-->计算真正用到的元素：

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

代码创建了一个表示[斐波那契数列](https://zhuanlan.zhihu.com/p/26752744)的延迟序列，<!--
-->它可以是无限长的<!--
-->（类似 [Haskell 的无限长列表](http://www.techrepublic.com/article/infinite-list-tricks-in-haskell/)）。<!--
-->我们可以只计算其中一些，例如，通过 `take()`：

```kotlin
println(fibonacci.take(10).joinToString())
```

> 这会打印出 `1, 1, 2, 3, 5, 8, 13, 21, 34, 55`。<!--
-->你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/fibonacci.kt)试一下。

生产者的优势在于支持任意的控制流，包括但不限于 `while`、<!--
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

### 异步用户界面

典型的有用户界面的应用程序只有一个事件调度线程，所有界面操作都发生在这个线程上。<!--
-->通常不允许在其他线程修改界面状态。所有用户界面库都提供<!--
-->某种原语，以将操作挪回界面线程中执行。例如，Swing 的 <!--
-->[`SwingUtilities.invokeLater`](https://docs.oracle.com/javase/8/docs/api/javax/swing/SwingUtilities.html#invokeLater-java.lang.Runnable-)，<!--
-->JavaFX 的 <!--
-->[`Platform.runLater`](https://docs.oracle.com/javase/8/javafx/api/javafx/application/Platform.html#runLater-java.lang.Runnable-)，<!--
-->Android 的 <!--
-->[`Activity.runOnUiThread`](https://developer.android.com/reference/android/app/Activity.html#runOnUiThread(java.lang.Runnable)) <!--
-->等等。<!--
-->下面是一个典型的 Swing 应用程序的代码片段，它执行一些异步<!--
-->操作，然后在用户界面中显示其结果:

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

这很像我们之间在[异步计算](#异步计算)用例见过的回调地狱，<!--
-->所以也能通过协程优雅地解决：

```kotlin
launch(Swing) {
    try {
        // 执行异步请求时挂起
        val result = makeRequest()
        // 在界面上显示结果，Swing 上下文保证了我们呆在事件调度线程上
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

* *协程* —— *可挂起计算* 的*实例*。它在概念上类似于线程，在这个意义上，<!--
  -->它需要一个代码块运行，并具有类似的生命周期 —— 它可以被*创建* 和*启动*，但它不绑定到<!--
  -->任何特定的线程。它可以在一个线程中*挂起* 其执行， 并在另一个线程中*恢复* 。<!--
  -->而且，像 future 或 promise 那样，它在*完结* 时可能伴随着某种结果（值或异常）。

* *挂起函数* —— `suspend` 修饰符标记的函数。它可能会通过调用其他挂起函数*挂起* 执行代码，<!--
  -->而不阻塞当前执行线程。挂起函数<!--
  -->不能在常规代码中被调用，只能在其他挂起函数或挂起 lambda 表达式中（见下方）。<!--
  -->例如，[用例](#用例)所示的 `.await()` 和 `yield()` 是<!--
  -->在库中定义的挂起函数。标准库提供了基础的挂起函数，用于定义<!--
  -->其他所有挂起函数。

* *挂起 lambda 表达式* —— 必须在协程中运行的代码块。<!--
  -->它看起来很像一个普通的 [lambda 表达式](https://kotlinlang.org/docs/reference/lambdas.html)，<!--
  -->但它的函数类型被 `suspend` 修饰符标记。<!--
  -->就像常规 lambda 表达式是匿名局部函数的短语法形式一样，<!--
  -->挂起 lambda 表达式是匿名挂起函数的短语法形式。它可能会通过调用其他挂起函数*挂起* 执行代码，<!--
  -->而不阻塞当前执行线程。<!--
  -->例如，[用例](#用例)所示的跟在 `launch` , `future` , 和 `BuildSequence` 函数后面花括号里的代码块<!--
  -->就是挂起 lambda 表达式。

  > 注意：挂起 lambda 表达式可以在其代码的任意位置调用挂起函数，只要这个位置能写从这个 lambda 表达式<!--
  -->[非局部](https://kotlinlang.org/docs/reference/returns.html) `return` 的语句。<!--
  -->也就是说，可以在像 [`apply{}` 代码块](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html)<!--
  -->这样的内联 lambda 表达式中调用挂起函数，<!--
  -->但在 `noinline` 和 `crossinline` 修饰的 lambda 表达式中就不行。<!--
  -->*挂起* 会被视作是一种特殊的非局部控制转移。

* *挂起函数类型*  —— 表示挂起函数和挂起 lambda 表达式的函数类型。它就像<!--
  -->一个一般的[函数类型](https://kotlinlang.org/docs/reference/lambdas.html#function-types)，<!--
  -->但具有 `suspend` 修饰符。举个例子，`suspend () -> Int` 是<!--
  -->一个没有参数、返回 `Int` 的挂起函数的函数类型。一个声明为 `suspend fun foo()  : Int` 的挂起函数<!--
  -->符合上述函数类型。

* *协程构建器* —— 使用一些挂起 lambda 表达式作为参数来创建一个协程的函数，<!--
  -->可能还提供某种形式以访问协程的结果。例如，[用例](#用例)中的 `launch{}`、  `future{}` <!--
  -->以及 `sequence{}` 就是协程构建器。<!--
  -->标准库提供了用于定义其他所有协程构建器所使用的基础协程构建器。

  > 注意：一些语言通过对特定方法的硬编码支持协程的创建、启动、<!--
  -->定义其执行的方式以及结果的表示方式。例如，`generate` *关键字* 可以定义<!--
  -->返回某种可迭代对象的协程，而 `async` *关键字* 可以定义<!--
  -->返回某种约定或任务的协程。Kotlin 没有关键字或修饰符来定义和启动协程。<!--
  -->协程构建器只是库中定义的简单的函数。<!--
  -->其他语言中以方法体形式定义的协程，<!--
  -->在 Kotlin 中，这样的方法通常是具有表达式方法体的普通方法，<!--
  -->方法体的内容是调用一个库中定义的、最后一个参数是挂起 lambda 表达式的协程构建器：

  ```kotlin
  fun doSomethingAsync() = async { ... }
  ```

* *挂起点* —— 协程执行过程中*可能被挂起* 的位置。<!--
  -->从语法上说，挂起点是对一个挂起函数的调用，但*实际* <!--
  -->的挂起在挂起函数调用了标准库中的原始挂起函数时发生。

* *续体* —— 是挂起的协程在挂起点时的状态。它在概念上代表<!--
  -->它在挂起点之后的剩余应执行的代码。例如：

  ```kotlin
  sequence {
      for (i in 1..10) yield(i * i)
      println("over")
  }  
  ```

  这里，每次调用挂起函数 `yield()`时，协程都会挂起，<!--
  -->*其执行的剩余部分* 被视作续体，所以有 10 个<!--
  -->续体：循环运行第一次后，`i=2`，挂起；循环运行第二次后，`i=3`，挂起……<!--
  -->最后一次打印 "over" 并完结协程。该协程在此*创建*，但尚未<!--
  -->*启动* 时，由它的*初始续体* 表示，这由它的整个执行组成，<!--
  -->类型为 `Continuation<Unit> ` 。

如上所述，驱动协程的要求之一是灵活性：<!--
-->我们希望能够支持许多现有的异步应用程序接口以及其他用例，并尽量减少<!--
-->硬编码到编译器中的部分。因此，编译器只负责支持<!--
-->挂起函数、挂起 lambda 表达式和相应的挂起函数类型。<!--
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
-->用户定义上下文。`resumeWIth` 函数是一个*完结* <!--
-->回调，用于报告协程完结时成功（带有值）或失败（带有异常）的结果。

为了方便，包里还定义了两个扩展函数：

```kotlin
fun <T> Continuation<T>.resume(value: T)
fun <T> Continuation<T>.resumeWithException(exception: Throwable)
```

### 挂起函数

一个典型的*挂起函数* 的某种实现，例如 `.await()` ，起来是这样的：

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

> 属性的取值器和设值器、构造器以及某些操作符函数<!--
-->（也就是 `getValue`，`setValue`，`provideDelegate`，`get`，`set` 以及 `equals`）不能带有 `suspend` 修饰符。<!--
-->这些限制将来可能会被消除。

挂起函数可以调用任何常规函数，但要真正挂起执行，必须<!--
-->调用一些其他的挂起函数。特别是，这个 `await` 实现调用了<!--
-->在标准库中定义的顶层挂起函数 <!--
-->[`suspendCoroutine`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html)<!--
-->（位于 `kotlinx.coroutines` 包）：

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

当  `suspendCoroutine` 在一个协程中被调用时（它*只* 可能在协程中<!--
-->被调用，因为它是一个挂起函数），它捕获了协程的执行状态<!--
-->到一个*续体* 实例，然后将其传给指定的 `block` 作为参数。<!--
-->为了恢复协程的执行，代码块需要在该线程或稍后在其他某个线程中调用 `continuation.resumeWith()`<!--
-->（直接调用或通过 `continuation.resume()` 或 `continuation.resumeWithException()` 调用）。<!--
--><!--
-->*实际* 的协程挂起发生在当 `suspendCoroutine` 代码块没有调用 `resumeWith` 就返回时。<!--
-->如果协程还未从代码块返回就直接被恢复，<!--
-->协程就不被看作已经暂停又继续执行。

传给 `continuation.resumeWith()` 的值作为调用 `suspendCoroutine` 的结果，<!--
-->进一步成为 `.await()` 的结果。

不允许多次恢复同一个协程，并会产生 `IllegalStateException`。 

> 注意：这正是 Kotlin 协程与像 Scheme 这样的函数式语言中的顶层限定续体<!--
-->或 Haskell 中的续体单子的关键区别。我们选择仅支持续体恢复一次，<!--
-->完全是出于实用主义考虑，因为所有这些预期的[用例](#用例)都不需要多重续体。<!--
-->然而，还是可以在另外的库中实现多重续体，<!--
-->通过底层的所谓[协程内建函数](#协程内建函数)复制续体中<!--
-->捕获的协程状态，然后就可以从这个副本再次恢复协程。

### 协程构建器

挂起函数不能够从常规函数中调用，所以标准库提供了<!--
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
-->函数（来自 `kotlin.coroutines` 包），它提供了一种<!--
-->简写以实现包含其给定的 `context` 值和 `resumeWith` 函数体的<!--
--> `Continuation` 接口。这个续体作为*完结续体* 被传给 <!--
-->[`block.startCoroutine(...)`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html) 扩展函数<!--
-->（来自 `kotlin.coroutines` 包）。

协程在完结时将调用其*完结续体*。其 `resumeWith` <!--
-->函数将在协程因成功或失败而*完结* 时调用。<!--
-->因为 `launch` 是那种“即发即弃”式的<!--
-->协程，它被定义成返回 `Unit` 的挂起函数，实际上是无视了<!--
-->其 `resume` 函数的结果。如果协程因异常完结，<!--
-->当前线程的未捕获异常句柄将用于报告这个异常。

> 注意：这个简单实现返回了 `Unit` ，根本不提供对协程状态的任何访问。<!--
-->实际上在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实现要更加<!--
-->复杂，因为它返回了一个 `Job` 接口的实例，代表这个协程，而且可以被取消。

上下文在[协程上下文](#协程上下文)一节中详细介绍。<!--
-->`startCoroutine` 在标准库中是无参和<!--
-->单参数的挂起函数类型的扩展函数：

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
fun <R, T> (suspend  R.() -> T).startCoroutine(receiver: R, completion: Continuation<T>)
```

`startCoroutine` 创建协程并在当前线程中立刻启动执行（但请参阅下面的备注），<!--
-->直到第一个*挂起点* 时返回。<!--
-->挂起点是协程中某个[挂起函数](#挂起函数)的调用，<!--
-->由相应的挂起函数的代码来定义协程恢复的时机和方式。

> 注意：续体拦截器（来自上下文）在[后文](#续体拦截器)中会提到，它能够<!--
-->将协程的执行，*包括* 其初始续体的执行，调度到另一个线程中。

### 协程上下文

协程上下文是一组可以附加到协程中的持久化用户定义对象。<!--
-->它可以包括负责协程线程策略的对象，日志，关于协程执行的安全性和事务方面的对象，<!--
-->协程的标识和名称等等。下面是协程及其上下文的简单认识模型。<!--
-->把协程看作一个轻量线程。在这种情况下，协程上下文就像是一堆线程局部化变量。<!--
-->不同之处在线程局部化变量是可变的，协程上下文是不可变的，<!--
-->但对于协程这并不是一个严重的限制，因为他们是如此轻量<!--
-->以至于当需要改变上下文时可以很容易地开一个新的协程。

标准库没有包含上下文的任何具体实现，<!--
-->但是有接口和抽象类，以便以*可组合* 的方式<!--
-->在库中定义所有这些方面，因此来自不同库的各个方面可以<!--
-->和平共存在同一个上下文中。

从概念上讲，协程上下文是一组可索引的元素，其中每个元素有唯一的键。<!--
-->它是集合和映射的混合体。它的元素有像在映射中的那样的键，但它的键直接与元素关联，<!--
-->更像在集合中。标准库定义了 <!--
-->[`CoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html) 的最小接口<!--
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
-->其他库定义的 `threadPool` 对象带着一些协程执行信息，<!--
-->你就可以使用[协程构建器](#协程构建器) `launch{}` 建造使用组合上下文的 <!--
-->`launch(auth + CommonPool){...}` 调用。

> 注意： [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 提供了几个上下文元素，<!--
-->包括用于在一个共享后台线程池中调度协程的 `Dispatchers.Default`。

标准库提供 <!--
-->[`EmptyCoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html) <!--
-->—— 一个不包括任何元素的（空的） `CoroutineContext` 实例。

所有第三方协程元素应该继承标准库的 <!--
-->[`AbstractCoroutineContextElement`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-abstract-coroutine-context-element/index.html) <!--
-->类（位于 `kotlinx.coroutines` 包）。<!--
-->要在库中定义上下文元素，建议使用以下风格。<!--
-->下面的这个例子显示了一个假想的储存当前用户名的授权上下文元素：

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

让我们回想一下[异步用户界面](#异步用户界面)用例。异步界面应用程序必须保证<!--
-->协程程序体始终在界面线程中执行，尽管事实上各种挂起函数<!--
-->是在任意的线程中恢复协程执行。这是使用*续体拦截器* 完成的。<!--
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
-->协程*挂起*，一段时间后按照相应挂起函数的定义，协程*恢复* 并执行 <!--
-->`block1`，接着再次挂起又恢复后执行 `block2`，在此之后协程*完结* 了。

续体拦截器可以选择拦截并包装 <!--
-->与 `initialCode`，`block1` 和 `block2` 执行相对应的、从它们恢复的位置到下一个挂起点之间的续体。 <!--
-->协程的初始化代码被视作是 <!--
-->由协程的*初始续体* 恢复得来。标准库提供了  <!--
-->[`ContinuationInterceptor`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html)  <!--
-->接口（位于 `kotlinx.coroutines` 包）：

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    fun releaseInterceptedContinuation(continuation: Continuation<*>)
}
```

`interceptContinuation` 函数包装了协程的续体。每当协程被挂起时，<!--
-->协程框架用下行代码包装实际后续恢复的 `continuation`：


```kotlin
val intercepted = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```

协程框架为每个实际的续体实例实例缓存拦截过的续体，并且在不再需要时<!--
-->调用 `releaseInterceptedContinuation(intercepted)`。<!--
-->想了解更多细节请参阅[实现细节](#实现细节)一节。

> 注意，像 `await` 这样的挂起函数实际上不一定会挂起协程的执行。<!--
  -->例如，[挂起函数](#挂起函数)一节所展现的 `await` 实现<!--
  -->在 future 已经完结的情况下就不会使协程真正挂起（在这种情况下 `resume` 会立刻被调用，<!--
  -->协程的执行并没有被挂起）。只有协程执行中真正被挂起时，续体才会被拦截，<!--
  -->也就是调用 `resume` 之前 `suspendCoroutine` 函数就返回了的时候。


让我们来看看 `Swing` 拦截器的具体示例代码，它将执行调度到 <!--
-->Swing 用户界面事件调度线程上。我们先来定义一个包装类 `SwingContinuation`，<!--
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

> 在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中，Swing上下文的实际实现<!--
  -->更加复杂，因为它还要集成库的计时和调试工具。

### 限定挂起

为了实现[生产者](#生产者)用例中的 `sequence{}` 和 `yield()`，需要另一类协程构建器和挂起函数。<!--
-->这是协程构建器 `sequence{}` 的示例代码：

```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

它使用了标准库中类似于 `startCoroutine`（解释见[协程构建器](#协程构建器)一节）的另一个原语 <!--
-->[`createCoroutine`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/create-coroutine.html)。<!--
--><!--
-->不同点在于它*创建* 一个协程，但并*不* 启动协程，<!--
-->而是返回表示协程的*初始续体* 的 `Continuation<Unit>` 引用：

```kotlin
fun <T> (suspend () -> T).createCoroutine(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutine(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

另一个不同点是传递给构建器的*挂起 lambda 表达式* `block` 是<!--
-->具有 `SequenceScope<T>` 接收者的<!--
-->[扩展 lambda 表达式](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver)。<!--
-->`SequenceScope<T>` 接口提供了生产者代码块的*作用域*，其在库中定义如下：


```kotlin
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

为了避免生成多个对象，`sequence{}` 实现中定义了 `SequenceCoroutine<T>` 类，<!--
-->它同时实现了 `SequenceScope<T>` 和 `Continuation<Unit>`，因此它可以同时作为 <!--
-->`createCoroutine` 的 `receiver` 参数和 `completion` 续体参数。<!--
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

    // 实现生产者
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequence.kt)看到代码。<!--
  -->注意，标准库提供了 <!--
  -->[`sequence`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/sequence.html) 函数开箱即用的优化实现<!--
  -->（位于 `kotlinx.coroutines` 包），<!--
  -->而且还支持 [`yieldAll`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence-scope/yield-all.html) 函数。

> 实际上的 `sequence` 代码使用了实验性的 `BuilderInference` 特性以支持<!--
  -->[生产者](#生产者)一节中使用的不用显式指定序列类型参数 `T` 的 `fibonacci` 声明。<!--
  -->其类型是从传给 `yield` 的参数类型推断得来的。

`yield` 的实现中使用了 `suspendCoroutine` [挂起函数](#挂起函数)来挂起。<!--
-->协程并捕获其续体。续体保存在 `nextStep` 中，。<!--
-->并在调用 `computeNext` 时恢复。

然而，之前展示的 `sequence{}` 和 `yield()`，其续体并不能被任意的挂起函数。<!--
-->在各自的作用域里捕获。它们*同步地* 工作。。<!--
-->它们需要对如何捕获续体、。<!--
-->在何处存储续体和何时恢复续体保持绝对的控制。它们形成了*限定挂起域*。<!--
-->对挂起的限定作用由作用域类或接口上的 `RestrictSuspension` 注解提供，。<!--
-->上面的例子里这个作用域接口是 `SequenceScope`：

```kotlin
@RestrictsSuspension
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

这个注解对能用在 `sequence{}` 域或其他类似的同步协程构建器中的挂起函数有一定的限制。<!--
--><!--
-->那些扩展*限定性挂起域* 类或接口<!--
-->（以 `@RestrictsSuspension` 标记）的<!--
-->挂起 lambda 表达式或函数称作*限定性挂起函数*。<!--
-->限定性挂起函数只接受来自同一个限定挂起域实例的的成员或扩展挂起函数作为参数。


回到这个例子，这意味着 <!--
-->`SequenceScope` 作用域的内扩展 lambda 表达式不能调用 `suspendContinuation` 或其他<!--
-->通用挂起函数。要挂起 `sequence` 协程的执行，最终必须通过调用 <!--
-->`SequenceScope.yield`。`yield` 本身被实现为 `SequenceScope` <!--
-->实现的成员函数，对其内部不作任何限制（只有*扩展* 挂起 lambda 表达式和函数是限定的）。

对于像 `sequence` 这样的限定性协程构建器，支持任意上下文是没有意义的，<!--
-->因为其作用类或接口（比如这个例子里的 `SequenceScope`）已经占用了上下文能提供的服务，因此限定性<!--
-->协程只能使用 `EmptyCoroutineContext` 作为上下文，`SequenceCouroutine` 的取值器实现<!--
-->也会返回这个。尝试创建上下文不是 `EmptyCoroutineSContext` 的限定性协程<!--
-->会引发 `IllegalArgumentException`。

## 实现细节

本节展现了协程实现细节的冰山一角。它们隐藏在[协程概述](#协程概述)部分解释的构造代码块背后，内部类和代码的生成策略在不打破公共 API 和 ABI 约定 的情况下随时可能改变。

### 续体传递风格

挂起函数通过 Continuation-Passing-Style (CPS) 实现。每个挂起函数和挂起 lambda 表达式都会在调用时隐式地传入一个附加的 `Continuation` 参数。回想一下，[`await` 挂起函数](#挂起函数) 的声明是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

然而在*CPS 变换*之后，它的实际实现具有具有以下签名：

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

其返回类型 `T` 移到了附加续体参数的类型参数位置。实现中的返回值 `Any?` 是用于代表挂起函数的动作。当挂起函数*挂起* 协程时，函数返回一个特别的标识值 `COROUTINE_SUSPENDED`（更多细节访问[`协程内建函数`](#协程内建函数)一节）。当一个挂起函数没有挂起协程，协程继续执行时，它直接返回一个结果或者抛出一个异常。这样以来，`await` 函数实现中的返回值 `Any?` 实际上是 `T` 和  `COROUTINE_SUSPENDED` 的联合类型，这并不能在 Kotlin 的类型系统中表示出来。

挂起函数的实现实际上不允许调用其栈帧中的续体，因为这可能导致长时间运行的协程栈溢出。标准库中的 `suspendCoroutine` 函数通过追踪续体的调用保证了无论续体在何时怎样调用，挂起函数与实际实现具有一致性的约定，得以对应用开发者隐藏它的复杂性。

### 状态机

协程实现的性能是非常重要的，这需要尽可能少地创建类和对象。许多语言通过*状态机* 实现，Kotlin 也是这样做的。对于 Kotlin，使用此方法让编译器为每个挂起 lambda 表达式创建一个其中可能有任意数量挂起点的类。

主要思想：挂起函数编译为状态机，其状态对应着挂起点。示例：弄一个有两个挂起点的挂起代码块：

```kotlin
val a = a()
val y = foo(a).await() // 挂起点 #1
b()
val z = bar(a, y).await() // 挂起点 #2
c(z)
```

这个代码块有三个状态：

- 初始化（在挂起点之前）
- 在第一个挂起点之后
- 在第二个挂起点之后

每个状态都是这个代码块续体的一个入口点（初始续体从第一行开始）。

代码会被编译为一个匿名类，它的一个方法实现了这个状态机、一个字段持有状态机当前状态，状态之间共享的局部变量字段（也可能有协程闭包的字段，<u>但在这种情况下它是空的</u>）。这是上文代码块通过续体传递风格调用挂起函数 `await` 的 Java 伪代码：

> <u>但在这种情况下它是空的</u>  
> 译者注：未能 get 到 `它` 指代什么。

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
        // 这次调用，result应该为空
        a = a()
        label = 1
        result = foo(a).await(this) // 'this'作为续体传递
        if (result == COROUTINE_SUSPENDED) return // 如果await挂起了执行则返回
      L1:
        // 外部代码传入.await()的结果恢复协程 
        y = (Y) result
        b()
        label = 2
        result = bar(a, y).await(this) // 'this'作为续体传递
        if (result == COROUTINE_SUSPENDED) return // 如果await挂起了执行则返回
      L2:
        // 外部代码传入.await()的结果恢复协程
        Z z = (Z) result
        c(z)
        label = -1 // 没有其他步骤了
        return
    }          
}    
```

请注意，这里有 `goto` 运算符，还有标签，因为该例子描述的变化发生在字节码中而不是源码中。

现在，当协程开始时，我们调用了它的 `resumeWith()` —— `label` 是 `0`，然后我们跳去 `L0`，接着我们做一些工作，将 `label` 设为下一个状态 —— `1`，调用 `.await()`，如果协程执行挂起就返回。当我们想继续执行时，我们再次调用 `resumeWith()`，现在它继续到了 `L1`，做一些工作，将状态设为 `2`，调用 `.await()`，同样在挂起时返回。下一次它从 `L3` 继续，将状态设为 `-1`，这意味着 "结束了，没有更多工作要做了"。

循环内的挂起点只生成一个状态，因为循环也通过 `goto` 工作（根据条件）：

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
        result = nextNumber().await(this) // 'this'作为续体传递 
        if (result == COROUTINE_SUSPENDED) return // 如果await挂起了执行则返回
      L1:
        // 外部代码传入.await()的结果恢复协程
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

挂起函数的字节码与它何时、怎样调用其他挂起函数有关。最简单的情况是一个挂起函数只在其*末尾* 调用其他挂起函数，这称作对它们的*尾调用*。对于那些实现低级并发原语或者包装回调函数的协程来说，这是典型的情况，就像[挂起函数](#挂起函数)一节和[包装回调](#包装回调)一节展示的那样。这些函数在末尾像调用 `suspendCoroutine` 那样调用其他挂起函数。编译这种挂起函数就和编译普通的非挂起函数一样，唯一的区别是通过 [CPS 转换](#协程传递样式)拿到的隐式续体参数会在尾调用中传递给下一个挂起函数。

如果挂起调用出现的位置不是末尾，编译器将为挂起函数生成 一个[状态机](#状态机)。状态机的实例在挂起函数调用时创建，在挂起函数完结时丢弃。

> 注意：以后的版本中编译策略可能会优化成在第一个挂起点生成状态机实例。

反过来，这个状态机实例成了其他非尾调用的挂起函数的*完结续体*。挂起函数多次调用其他挂起函数时，状态机实例会被更新并重用。对比其他[异步编程风格](#异步编程风格)，异步过程的每个后续步骤通常使用单独的、新分配的闭合对象。

### 协程内建函数

>  TODO

## 附录

这不是一个规范的章节，没有引入新的语言结构或库函数，而是讨论了一些涉及资源管理、并发和编码风格的话题，并为各种各样的用例提供了更多示例。

### 资源管理与垃圾收集

协程不使用堆外存储，也不自行消耗任何本机资源，除非在协程中运行的代码打开了文件或占用了其他资源。在协程中打开的文件必然要以某种方式关闭，这不意味着协程本身需要关闭。当协程挂起时，其状态可以通过对其续体的引用来获取。如果你失去了对挂起协程续体的引用，最终它会被垃圾收集器回收。

打开了可关闭资源的协程应该特别关注。考虑下面这个[受限挂起](#受限挂起)一节中使用 `sequence{}` 构建器从文件生成行序列的协程：

```kotlin
fun sequenceOfLines(fileName: String) = sequence<String> {
    BufferedReader(FileReader(fileName)).use {
        while (true) {
            yield(it.readLine() ?: break)
        }
    }
}
```

这个函数返回一个 `Sequence<String>`，通过这个函数，你可以用一种自然的方式打印文件的所有行：

```kotlin
sequenceOfLines("https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt")
    .forEach(::println)
```

> 从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt)获取完整代码

只要你遍历 `sequenceOfLines` 函数返回的整个序列，它就工作正常。然而，如果你只打印了文件的前几行，就像这样：

```kotlin
sequenceOfLines("https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequenceOfLines.kt")
        .take(3)
        .forEach(::println)
```

协程恢复了几次，产生出文件的前三行，然后就被*遗弃* 了。遗弃对于协程本身来说没什么关系，但是对于打开了的文件则不然。[`use` 函数](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html)没有机会结束调用并关闭文件。文件会一直开着，直到被垃圾收集器回收，因为 Java 的文件操作有个 `finalizer` 能关闭文件。如果只是个幻灯片或者短时间运行的小工具，这倒也不是什么问题，但是对于那些大型后端系统来说可就是个灾难了，因为它们有几个GB的堆存储，以至于在耗尽内存触发垃圾收集之前文件句柄先溢出了。

这个问题和 Java 里生成行的惰性流的 [`Files.lines`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#lines-java.nio.file.Path-) 遇到的问题一样。它返回一个可关闭的 Java 流，但多数流操作不会自动调用对应的 `stream.close` 方法，需要用户自己记着关闭流。Kotlin 里也可以定义可关闭序列的生产者，当然也会遇到同一个问题，就是语言没有什么自动机制能保证它们在用完之后关闭。引入一种自动化资源管理的语言机制明显超出了 Kotlin 协程的领域。

然而，通常这个问题不会影响协程的异步用例。异步协程是不会被遗弃的，它会持续运行直到完毕。因此只要协程里的代码能正确地关闭其资源，资源最终就会被关闭。

### 并发和线程

在一个独立协程的内部，如同线程内部一样，是顺序执行的。这意味着下面这种协程内的代码是相当安全的：

```kotlin
launch { // 启动协程
    val m = mutableMapOf<String, String>()
    val v1 = someAsyncTask1() // 开始一些异步任务
    val v2 = someAsyncTask2() // 开始一些异步任务
    m["k1"] = v1.await() // 修改映射等待操作完成
    m["k2"] = v2.await() // 修改映射等待操作完成
}
```

在协程的作用域里，你可以随意使用那些普通的非线程安全可变的结构。然而，在协程*之间* 共享可变状态仍可能带来致命威胁。如果你使用了一个指定调度器的协程构建器，在单一事件调度线程上恢复协程，就像[协程拦截器](#协程拦截器)一节展示的 `Swing` 拦截器那样，那你仍能安全地操作所有共享对象，因为它们总在事件调度线程上修改。但如果你在多线程环境中或者需要在运行在不同线程上的多个协程之间共享可变状态，你就必须使用线程安全的（并发的）数据结构。

协程在这方面和线程没有什么不同，尽管协程确实更轻。你可以在仅仅几个线程上同时运行几百万个协程。一个运行着的协程总是在某个线程上。但一个*挂起* 了的协程并不占用线程，也没有以任何方式绑定到线程。恢复协程的挂起函数通过在线程上调用 `Continuation.resumeWith` 决定在哪个线程上恢复协程。而协程拦截器可以覆盖这个决定，并将协程的执行分派到不同的线程上。

### 异步编程风格

异步编程有多种风格。

[异步计算](#异步计算)一节已经讨论了回调函数，这也是协程风格通常用来替换的最不方便的一种风格。任何回调风格的应用程序接口都可以用对应的挂起函数包装，见[这里](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#wrapping-callbacks)。

我们来回顾一下。假设你现在有一个带有以下签名的阻塞 `sendMail` 函数：

```kotlin
fun sendEmail(emailArgs: EmailArgs): EmailResult
```

它在运行时会阻塞执行线程很长的时间。

要使其不阻塞，可以使用错误先行的 [node.js 回调约定](https://www.tutorialspoint.com/nodejs/nodejs_callbacks_concept.htm)，以回调样式表示其非阻塞版本，签名如下：

```kotlin
fun sendEmail(emailArgs: EmailArgs, callback: (Throwable?, EmailResult?) -> Unit)
```

但是，协程支持其他风格的异步非阻塞编程。其中之一是内置于许多流行语言中的 async/await 风格。在Kotlin中，可以通过引入 `future{}` 和 `.await()` 库函数来复制这种风格，就像 [futures](#Futures) 用例一节所示。

这种风格主张从函数返回对未来对象的某种约定，而不是传入回调函数作为参数。在这种异步风格中，`sendEmail` 的签名看起来是这样：

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult>
```

从风格上讲，最好给这方法的名字加上一个 Async 后缀，因为它们的参数和阻塞版本没什么不同，因而很容易忘记其操作异步的本质。函数 `sendEmailAsync` 启动一个*并发* 异步的操作，可能带来并发的所有陷阱。然而，鼓励这种风格的编程语言通常也提供某种 `await` 原语，在需要的时候把操作重新变回顺序的。

Kotlin 的*原生* 编程风格基于挂起函数。在这种风格下，`sendEmail` 的签名看起来比较自然，不破坏其参数或返回类型，但带有附加的 `suspend` 修饰符：

```kotlin
suspend fun sendEmail(emailArgs: EmailArgs): EmailResult
```

我们已经发现，异步和挂起风格可以通过原语很容易地相互转换。例如，`sendEmailAsync` 可以用挂起版本的 `sendEmail` 和 [future 构建器](#https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#building-futures)轻易地实现：

```kotlin
fun sendEmailAsync(emailArgs: EmailArgs): Future<EmailResult> = future {
    sendEmail(emailArgs)
}
```

因此，在某种意义上，这两种样式是等效的，并且在方便性上都明显优于回调样式。但是，让我们更深入地研究 `sendEmailAsync` 和挂起 `sendEmail` 之间的区别。

让我们先比较一下他们在代码中调用的方式。挂起函数可以像普通函数一样调用：

```kotlin
suspend fun largerBusinessProcess() {
    // 这里有很多代码，接下来在某处……
    sendEmail(emailArgs)
    // ……后来又继续做了些别的事
}
```

对应的异步风格函数这样调用：

```kotlin
fun largerBusinessProcessAsync() = future {
    // 这里有很多代码，接下来在某处……
    sendEmailAsync(emailArgs).await()
    // ……后来又继续做了些别的事
}
```

注意，异步风格的函数调用写法更冗长，更容易出错。如果在异步风格的示例中省略了 `.await()` 调用，代码仍然可以编译并工作，但现在它将异步发送电子邮件，甚至在执行较大业务流程的其余部分的同时发送电子邮件，因此可能会修改某些共享状态并引入一些非常难以重现的错误。相反，挂起函数默认是顺序的。对于挂起的函数，无论何时需要任何并发，都可以在代码中通过调用某种 `future{}` 或类似的协程构建器显式地表达。

从这些风格在使用多个库的大型项目中的**比例**方面比较。挂起函数是 Kotlin 的一个轻量级语言概念。所有挂起函数在任何非限定性的 Kotlin  协程中都是完全可用的。异步风格的函数依赖于框架。每个 promises/futures 框架都必须定义自己的类 `async` 函数，该函数返回自己的 promise/future 类，这些类又有对应的类 `async` 函数。

从**性能**方面比较。挂起函数提供最小的调用开销。你可以看[实现细节](#实现细节)一节。异步类型的函数对所有挂起机制需要额外持有相当重的 promise/future 抽象。异步样式的函数调用中必须返回一些 future 的实例对象，并且即使函数非常简短，也无法将其优化掉。异步样式不太适合于非常细粒度的分解。

从与 JVM/JS 代码的**互操作性**方面比较。使用对应类 future 抽象的异步风格函数与 JVM/JS 代码更具互操作性。在 Java 或 JS 中，它们只是返回相应类 future 对象的函数。对任何不原生支持[续体传递风格](#续体传递风格)的语言来说，挂起函数都很奇怪。但是从上面的示例中可以看出，对于任何给定的 promise/future 框架都很容易将任何挂起函数转换为异步风格的函数。因此，您只需用 Kotlin 编写一次挂起函数，然后使用适当的 `future{}` 协程构建器函数，通过一行代码对其进行调整，以实现与任何形式的 promise/future 的互操作性。

### 包装回调

很多异步应用程序接口集包含回调风格的接口。标准库中的挂起函数 `suspendCoroutine` （见[挂起函数](#挂起函数)一节）提供了一种简单的把任何回调函数包装成 Kotlin 挂起函数的方法。

这里有一个简单的例子。假设你有一个 `someLongCompution` 函数，他有一个回调参数，回调接受某种 `Value` 参数，这个 `Value` 是计算的结果。

```kotlin
fun someLongComputation(params: Params, callback: (Value) -> Unit)
```

你可以用下面这样的代码直截了当的把它变成挂起函数：

```kotlin
suspend fun someLongComputation(params: Params): Value = suspendCoroutine { cont ->
    someLongComputation(params) { cont.resume(it) }
} 
```

现在计算的返回值变成显式的了，但它还是异步的，也不会阻塞线程。

> 注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 包含了一个协作式可取消协程框架。它提供类似 `suspendCoroutine`，但支持取消的 `suspendCancellableCoroutine` 函数。查看其指南中[取消时](http://kotlinlang.org/docs/reference/coroutines/cancellation-and-timeouts.html)一节了解更多细节。

为了找一个更复杂的例子，我们看看[异步计算](#异步计算)用例中的 `aRead()` 函数。它可以实现为 Java 非阻塞输入输出中 [`AsynchronousFileChannel`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/AsynchronousFileChannel.html) 的挂起扩展函数，它的 [`CompletionHandler`](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/CompletionHandler.html) 回调接口如下：

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

> 从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/io/io.kt)获取代码。注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中实际的实现支持取消以放弃长时间运行的输入输出操作。

如果你需要处理大量有相同类型回调的函数，你可以定义一个公共包装函数简便地把他们全部转换成挂起函数。例如，[vert.x](http://vertx.io/) 有一个特有的约定，其中所有异步函数都接受一个 `Handler<AsyncResult<T>>` 回调。要通过协程简化任意的 vert.x 函数，可以定义下面这个辅助函数：

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

通过这个辅助函数，任意异步 vert.x 函数 `async.foo(params, handler)` 可以在协程中这样调用：`vx { async.foo(params, it) }`。

### 构建 Future

定义在 [future](#Future) 用例中类似于 `launch{}` 构建器的 `future{}` 构建器可以用于实现任何 future 或 promise 原语，这在[协程构建器](#协程构建器)做了一些介绍：

```kotlin
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
        CompletableFutureCoroutine<T>(context).also { block.startCoroutine(completion = it) }
```

它与 `launch{}` 的第一点不同是它返回 [`CompletableFuture`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 的实例，第二点不同是它包含一个默认为 `CommonPool` 的上下文，因此其默认执行在 [`ForkJoinPool.commonPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--)，这个默认执行行为类似于 [`CompletableFuture.supplyAsync`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) 方法。`CompletableFutureCoroutine` 的基本实现很直白：

```kotlin
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
    override fun resumeWith(result: Result<T>) {
        result
            .onSuccess { complete(it) }
            .onFailure { completeExceptionally(it) }
    }
}
```

> 从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/future/future.kt)获取代码。[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中实际的实现更高级，因为它要传播对等待结果的期货的取消，以终止协程。

协程完结时调用对应 future 的 `complete` 方法向协程报告结果。

### 非阻塞睡眠

协程不应使用 [`Thread.sleep`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-)，因为它阻塞了线程。但是，通过 Java 的 [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html) 实现挂起的非阻塞 `delay` 函数是非常简单的。

```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
    Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont ->
    executor.schedule({ cont.resume(Unit) }, time, unit)
}
```

> 你可以从 [这里](https://github.com/Kotlin/kotlin-coroutines/blob/master/examples/delay/delay.kt) 获取到这段代码。注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)  同样提供了 `delay` 函数。

注意，这种 `delay` 函数从其单独的“时刻表”线程恢复协程。那些使用[拦截器](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#continuation-interceptor)的协程，比如 `Swing`，不会在这个线程上执行，因为它们的拦截器在合适的线程上调度它们。没有拦截器的协程会在时刻表线程上调度。所以这对一个示例来说挺方便的，但是算不上有性能。最好能在相应的拦截器中实现原生的睡眠。对于 `Swing` 拦截器，非阻塞睡眠的原生实现应使用专门为此目的设计的 [`Swing 计时器`](https://docs.oracle.com/javase/8/docs/api/javax/swing/Timer.html)：

```kotlin
suspend fun Swing.delay(millis: Int): Unit = suspendCoroutine { cont ->
    Timer(millis) { cont.resume(Unit) }.apply {
        isRepeats = false
        start()
    }
}
```

> 你可以从 [这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/swing-delay.kt) 获取到这段代码。注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)  中的 `delay` 实现注意了拦截器的特异性睡眠机制，并在适当的情况下自动使用上述方法。

### 协作式单线程多任务

在单线程应用中实现多任务非常方便，因为这样就不必处理并发或者共享可变状态了。JS、Python 还有很多其他语言甚至没有线程，但有协作式多任务原语。

[协程拦截器](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#coroutine-interceptor)提供了一个简单的工具保证所有协程限制在同一线程上。[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/threadContext.kt)的示例代码定义了 `newSingleThreadContext()` 函数，它能创建一个单线程执行的服务并使其适应协程拦截器的需求。

在下面这个单线程的示例中，我们把它和[构造 futures](#构造 futures)一节中的 `future{}` 协程构建器一起使用，尽管它有两个同时处于活动状态的异步任务。

```kotlin
fun main(args: Array<String>) {
    log("启动事件线程")
    val context = newSingleThreadContext("事件线程")
    val f = future(context) {
        log("Hello, world!")
        val f1 = future(context) {
            log("f1 睡眠")
            delay(1000) // 睡眠1秒
            log("f1 返回 1")
            1
        }
        val f2 = future(context) {
            log("f2 睡眠")
            delay(1000) // 睡眠1秒
            log("f2 返回 2")
            2
        }
        log("等待 f1 和 f2 都完结，只需1秒！")
        val sum = f1.await() + f2.await()
        log("和是$sum")
    }
    f.get()
    log("结束")
}
```

> 从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/threadContext-example.kt)获取完整示例。注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)  有 `newSingleThreadContext` 开箱即用的实现。

如果你的整个应用都在同一个线程上执行，你可以定义自己的辅助协程构建器，在其中硬编码一个适应你单线程执行机制的上下文。

### 异步序列

[受限挂起](#受限挂起)一节提到的 `sequence{}` 协程构建器是一个*同步* 协程的示例。当消费者调用 `Iterator.next()` 时，协程的生产代码同步执行在同一个线程上。`sequence{}` 协程块是受限的，没法用第三方挂起函数挂起其执行，比如[包装回调](#包装回调)一节中那种异步文件输入输出。

*异步的* 序列构建器支持随意挂起和恢复执行。这意味着其消费者要时刻准备着处理数据还没生产出来的情况。这是挂起函数的原生用例。我们来定义一个类似于普通 [`Iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-iterator/) 接口的 `SuspendingIterator` 接口，但其 `next()` 和 `hasNext()` 函数是挂起的：

```kotlin
interface SuspendingIterator<out T> {
    suspend operator fun hasNext(): Boolean
    suspend operator fun next(): T
}
```

`SuspendingSequence` 的定义类似于标准 [`Sequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence/index.html) 但返回 `SuspendingIterator`：

```kotlin
interface SuspendingSequence<out T> {
    operator fun iterator(): SuspendingIterator<T>
}
```

就像同步序列的作用域一样，我们也给它定义一个作用域接口，但它的挂起不是受限的：

```kotlin
interface SuspendingSequenceScope<in T> {
    suspend fun yield(value: T)
}
```

构建器函数 `suspendingSequence{}` 的用法和同步的 `sequence{}` 一样。

它们的区别在于 `SuspendingIteratorCoroutine` 的实现细节以及在下面这种情况中，异步版本接受一个可选的上下文：

```kotlin
fun <T> suspendingSequence(
    context: CoroutineContext = EmptyCoroutineContext,
    block: suspend SuspendingSequenceScope<T>.() -> Unit
): SuspendingSequence<T> = object : SuspendingSequence<T> {
    override fun iterator(): SuspendingIterator<T> = suspendingIterator(context, block)
}
```

> 从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/suspendingSequence/suspendingSequence.kt)获取完整代码。注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中对 `Channel` 原语的实现使用了对应的协程构建器 `produce{}`，其中对同一个概念提供了更复杂的实现。

我们可以用上[单线程多任务](#单线程多任务)一节的 `newSingleThreadContext{}` 上下文和[非阻塞睡眠](#非阻塞睡眠)一节的非阻塞的 `delay` 函数。

这样我们就能写一个非阻塞序列的实现来生产1~10的整数，两数之间间隔500毫秒：

```kotlin
val seq = suspendingSequence(context) {
    for (i in 1..10) {
        yield(i)
        delay(500L)
    }
}
```

现在消费者协程可以按自己喜欢的方式消费协程了，也可以被任意的挂起函数挂起。 注意，Kotlin [for 循环](https://kotlinlang.org/docs/reference/control-flow.html#for-loops)的工作方式满足这种序列的约定，因此语言中不需要一个专门的 `await for` 循环结构。普通的 `for` 循环就能用来遍历我们刚刚定义的异步序列。生产者没有值的时候它就会挂起：

```kotlin
for (value in seq) { // 等待生产者生产时挂起
    // 在这里用值做些事，也可以在这里挂起
}
```

>   [这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/suspendingSequence/suspendingSequence-example.kt)有写好的示例，其中用一些日志表示要执行的操作。

### 通道

Go 风格的类型安全通道在 Kotlin 中通过库实现。我们可以为发送通道定义一个接口，包含挂起函数 `send`：

```kotlin
interface SendChannel<T> {
    suspend fun send(value: T)
    fun close()
}
```

以及风格类似[异步序列](#异步序列)的接收通道，包含挂起函数 `receive` 和 `operator iterator`：

```kotlin
interface ReceiveChannel<T> {
    suspend fun receive(): T
    suspend operator fun iterator(): ReceiveIterator<T>
}
```

`Channel<T>` 类同时实现这两个接口。通道缓存满时 `send` 挂起，通道缓存空时 `receive` 挂起。这样我们可以一字不差的复制 Go 风格的代码。[Go 教程的第4个并发示例](https://tour.golang.org/concurrency/4)中向通道发送 n 个斐波那契数的 `fibonacci` 函数用 Kotlin 实现起来是这样：

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

我们也可以定义 Go 风格的 `go {...}` 代码块在某种线程池上启动新协程，在固定数量的重量线程上调度任意多的轻量协程。[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/go.kt)的示例实现写在 Java 通用的的 [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html) 里。

使用 `go` 协程构建器，对应的 Go 代码主函数看起来是下面这样，其中的`mainBlocking` 是简化的辅助函数，它在 `go{}` 的线程池上调用 `runBlocking`：

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val c = Channel<Int>(2)
    go { fibonacci(10, c) }
    for (i in c) {
        println(i)
    }
}
```

> 在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-4.kt)查看代码

你可以随意修改通道的缓冲区容量。为了简化，例子中只实现了缓冲通道（最小缓冲1个值），因为无缓冲通道在概念上和我们刚才见过的[异步序列](#异步序列)一样。

Go 风格的 `select` 控制流，作用是挂起直到其中一个通道上的一个操作可以生效，可以用 Kotlin DSL 这样实现，因此 [Go 教程的第5个并发示例](https://tour.golang.org/concurrency/5)在 Kotlin 里看起来是这样：

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

> 在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-5.kt)查看代码

例子用到了 `select {...}` 实现，它选择一种情况并返回结果，就像 Kotlin 的 
[`when` 表达式](https://kotlinlang.org/docs/reference/control-flow.html#when-expression)。还用到了一个方便的 `whileSelect { ... }`，它就是 `while(select<Boolean> { ... })`，但需要的括号比较少。

实现 [Go 教程的第6个并发示例](https://tour.golang.org/concurrency/6)中的默认选项只需添加另一个选项到 `select {...}` DSL：

```kotlin
fun main(args: Array<String>) = mainBlocking {
    val tick = Time.tick(100)
    val boom = Time.after(500)
    whileSelect {
        tick.onReceive {
            println("tick.")
            true // continue loop
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

> 在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-6.kt)查看代码

 [这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/time.kt)的 `Time.tick` 和 `Time.after` 用非阻塞的 `delay` 函数实现非常简单。

[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/)能找到其他示例，注释里有对应的 Go 代码的链接。

注意，这是通道的简单实现，只用了一个锁来管理内部的等待队列。这使得它容易理解和解释。但是，它从不在这个锁下运行用户代码，因此它是完全并发的。这个锁只在一定程度上限制了它对大量并发线程的可伸缩性。

> 通道和 `select` 在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实际实现基于无锁的无冲突并发访问数据结构。

这样实现的通道不影响协程上下文中的拦截器。 它可以用于用户界面应用程序，通过[续体拦截器](#续体拦截器)一节提到的事件线程拦截器，或者任何别的拦截器，或者不使用任何拦截器也可以（在后一种情况下，实际的执行线程完全由协程中使用的其他挂起函数的代码决定）。通道实现提供的挂起函数都是非阻塞且线程安全的。

### 互斥

编写可伸缩的异步应用程序应遵循一个原则，确保代码挂起（使用挂起函数）而不阻塞，即实际上不阻塞线程。Java并发原语 [`ReentrantLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html) 阻塞线程，不应在真正的非阻塞代码中使用。要控制对共享资源的访问，可以定义一个 `Mutex` 类，该类挂起协程的执行，而不是阻塞协程。

这个类的声明看起来是这样：

```kotlin
class Mutex {
    suspend fun lock()
    fun unlock()
}
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/mutex/mutex.kt)获得完整的实现。在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实际实现还有其他的一些函数。

使用这个非阻塞互斥的实现，[Go教程的第9个并发示例](https://tour.golang.org/concurrency/9)可以用 Kotlin 的 [`try finally`](https://kotlinlang.org/docs/reference/exceptions.html) 翻译到 Kotlin，这与 Go 的 `defer` 作用相同：

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

> 在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/channel/channel-example-9.kt)查看代码

### 从实验性协程移植

> TODO

### 参考

> TODO

### 反馈

> TODO

## 名词对照

| 原文                       | 译文          | 语境                      |
| :------------------------: | :----------: | :-----------------------: |
| complete / completion      | 完结         | 主语是协程，或协程包装的东西 |
| continuation               | 续体         | 所有                       |
| continuation passing style | 续体传递风格  | 所有                       |
| coroutine intrinsics       | 协程内建函数  | 所有                       |
