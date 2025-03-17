# Java 线程与 Kotlin 协程的一些探讨

本文目前仍处于一篇草稿的水平，参考借鉴了许多高质量博文，以综述的形式总结了线程与协程的一些概念。

### 主要内容

- Java 线程的本质是什么？与真正操作系统的线程又有何关系？
- GoLang 的 Goroutine 与 Java 的线程相比有何差异？
- Kotlin 协程的本质是什么？与 Goroutine 的原理相同吗？

### Java Thread && 内核线程

Java Thread 的实现实际上是对操作系统内核线程的封装，并且在主流的 JVM 实现上，Java Thread 与操作系统内核线程是 1:1 映射的。这种用户空间线程与操作系统内核线程一对一地映射的设计，既有其优势，也有其不足之处。

优势在于，语言层面无需去实现对 Thread 的调度，无需关心任何平台相关的细节，依赖于操作系统的调度即可；这种跨平台的封装，让我们可以无需考虑任何底层细节，即可以方便的创建并使用操作系统级别的内核线程，当然，Thread 类经过封装，对开发者提供的接口实际上是有有限的，我们不具备许多较为底层的对线程的控制能力。

这种用户空间线程与操作系统内核线程一对一地映射、且线程调度依赖于操作系统的调度的设计，也带来了许多限制。

- 限制1: 由于 JVM 会为每条创建的线程分配一个较大的、固定大小的栈空间，便存在线程创建过多导致的内存溢出的风险；当然，我们可以修改这个栈空间的大小，但是栈空间若变小，又会存在栈溢出的风险。
- 限制2: 线程上下文切换是存在一定的耗时成本的，因此为了执行大量并发的异步任务而过多地创建线程时，是会带来线程切换的性能损耗的。

上述两条限制，导致了我们的 JVM 实际上无法创建过多的线程，并且随着线程创建地过多，会极大地影响到异步任务的运行效率.

这背后其实反应的问题是：在既有的操作系统之下，任何将语言层面的线程与操作级别内核线程进行 1:1 映射的异步并发机制都无法支持超大规模的并发.

那么为了支持更大规模的并发，一个很直观的解决思路便是设计一种“操作系统线程模型与轻量级、用户空间的线程模型之间多对多映射的模型”。实际上，现在许多主流的支持高效并发的语言或者并发框架，其都免不了使用这种优化思路。如，在 Apache 中，每个请求都是由一个内核线程来处理的，这限制了 Apache 只能有效处理数千个并发连接，但是 Nginx 则选择了另外一种模型，一个操作系统内核线程能够应对上百个甚至上千的并发连接，从而允许更高程度的并发；同样在语言层面的线程与操作系统内核线程一对一映射的 Python, 实际上也受到这种单一映射设计带来的限制，无法支持高效并发，但是 Gevent 的引入能够为其带来 greenlet 这一用户空间线程能力，使得 Python 能够支持更高程度的并发；GoLang 的协程（Goroutine），亦是一种这样的设计思路，它在语言层面实现了对协程的调度，并不依赖与线程的上下文切换，不同的协程可以运行在相同的线程之上，支持比 Java 更高效的并发。

### Goroutine && Kotlin Coroutine

提到“协程”，各类技术文章中对其最常使用的简易描述便是“更轻量级的线程”，这种描述形象地凸显了协程的设计初衷，即期待它来实现比线程更高效的并发性能。协程与线程确实有类似之处，它们都是能够线性地执行的一系列逻辑操作，但是差别在于，线程实际上是接受操作系统的调度的，而协程并不需要来自于操作系统的同步、调度支持，而协程的调度切换则不涉及到任何操作系统层面的调用或任何阻塞调用. 说的更加实际直白些，协程是这样一种代码块，它支持类似于线程的“挂起”与“恢复”，这种挂起与恢复操作并不需要依赖于操作系统提供的同步能力的支持。

目前主流的程序设计语言，对于协程都有不同程度的实现，其中 GoLang 的 Goroutine 的实现较为典型且十分完备。我们在这里借助一些参考资料对其设计与实现进行简易的讨论与总结。

Golang 中并没有类似 Java Thread 这种对操作系统级别的线程进行映射的工具提供给到用户空间供程序员使用，其提供了 Goroutine 以及在 Golang 语言层面实现的调度器作为其异步任务执行工具。Goroutine 只存在于 Go 语言的运行时，它是 Go 语言在用户态提供的线程，作为一种粒度更细的资源调度单元，如果使用得当能够在高并发的场景下更高效地利用机器的 CPU。

Gorotuine 就是 Go 语言调度器中待执行的任务，它在运行时调度器中的地位与线程在操作系统中差不多，但是相比线程，它占用了更小的内存空间，且得益于优秀的调度策略，协程之间的切换并不依赖于操作系统内核线程的切换，因此也大大也降低了上下文切换的开销。协程占用空间小，上下文切换开销小，恰巧像是分别应对上一节中提到的 JVM 线程模型的两点限制进行的改进。

[这份中文文档](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/) 对 Golang 的调度器实现历程进行了翔实的介绍，建议浏览阅读。

根据文档，如果不做特殊配置，Golang 运行时实际上只会创建出与系统内核数量相等的线程，而基于协作的抢占式调度器则能够高效地在这有限的线程上调度数量众多的协程的执行。

Kotlin 这门 JVM 上实现的方言，在 1.3 版本开始，正式对外提供了协程的支持。它是否如 Golang 一样，实现了与之类似的协程机制，并且能够在 Java 线程机制的基础上，实现更高性能的调度呢？

Kotlin 协程实际上提供了一部分“协程”的能力，即 Coroutine 能够作为一个比 Runnable 更小的代码块单元，被挂起、自动恢复，但是 Kotlin 并没有实现高效的调度策略去实现对协程的调度，受制于 JVM 的带有互斥代码块和条件变量的经典的面向线程的并发模型，Kotlin 协程的调度实际上仍旧是依赖于 JVM 的同步机制来实现的，也依旧是受到操作系统的调度的。

Kotlin 提供的 Coroutine 这一“协程库”，其本质上仍旧是对 Java 线程的一层封装，只是一个使用上较为方便的异步任务执行工具而已。就像 RxJava#Scheduler 的链式调用、亦或者 Android SDK 中提供的 AsyncTask 等异步任务执行工具。

Java 线程是接受操作系统调度的, Java 线程一旦被阻塞, 便无法被继续使用, 直到阻塞并且再次被操作系统分配到时间片. 一个Kotlin 协程转到后台执行本质上依赖的依旧是线程池机制, 作为一个可以异步执行的代码块, 它在启动的时候并不需要指定到某个固定的线程上, 协程本身可以在一条线程被挂起, 在另一条线程上被恢复. 其实和线程池机制是十分雷同的, 只是协程这种进一步的抽象封装, 可以在异步执行完成后, 自动恢复到原来的执行位置, 让其继续执行下去.

### Kotlin Coroutine 的意义

但是 Kotlin 协程库也并不是如一些网络上“拨乱反正”的文章一样写的，是一种“假协程”。协程本身就是一种概念抽象，Kotlin 确实在 JVM 现有机制的基础上，实现了具备协程基本能力的 Coroutine. 使用 Coroutine, 我们可以非常方便地、以同步风格代码编写异步代码。

### 线程与协程孰优孰劣？

二者其实各有优劣，抛开场景谈性能是没有实际意义的。但就从支持更高效并发的角度来看，协程这种设计模式肯定是更高效的。

### 如何看待响应式编程

Reactive 编程模型已经被业界广泛接受，是一种重要的技术方向；同时Java代码里的阻塞也很难完全避免。协程可以作为一种底层 worker 机制来支持 Reactive 编程，即保留了 Reactive 编程模型，也不用太担心用户代码的阻塞导致了整个系统阻塞。

### Ref

- [https://www.ibm.com/developerworks/cn/java/j-lo-processthread](https://www.ibm.com/developerworks/cn/java/j-lo-processthread)

    Java 进程与线程的底层实现.

- [https://stackoverflow.com/questions/46864623/which-of-coroutines-goroutines-and-kotlin-coroutines-are-faster](https://stackoverflow.com/questions/46864623/which-of-coroutines-goroutines-and-kotlin-coroutines-are-faster)
- [https://stackoverflow.com/questions/48106252/why-threads-are-showing-better-performance-than-coroutines](https://stackoverflow.com/questions/48106252/why-threads-are-showing-better-performance-than-coroutines)

    这两个 stackoverflow 上的回答让人有醍醐灌顶之感. 抛开场景谈性能本就是极不合理的, Thread 与 Coroutine 孰优孰劣是无法有定论的. 不能因为一句“协程是轻量级的线程”这种半误导似的的话, 就钻牛角尖式地、不论场景和实现地去探究二者的性能.

- [为什么Goroutine能有上百万个，Java线程却只能有上千个？](https://mp.weixin.qq.com/s/v-Q5aOnYVj7l-kMQopkPLA)
- 英文原文: [https://rcoh.me/posts/why-you-can-have-a-million-go-routines-but-only-1000-java-threads/](https://rcoh.me/posts/why-you-can-have-a-million-go-routines-but-only-1000-java-threads/)

    这篇文章从以下几个方面详细地阐述了为什么Goroutine能有上百万个，Java线程却只能有上千个？

        - 理解线程与协程；
        - 了解线程的本质；
        - 了解Java线程在底层的实现本质；
        - 懂得为什么需要有Go、Java等多种语言，他们不仅仅是语法上的不同，在许多底层实现上也有不同的差异，因此也造成了它们各自不同的应用场景。


- [https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)

    这份在线文档的“6.5调度器”一章，详细介绍了 Go 语言对协程调度策略的基本实现原理.

- [https://studygolang.com/articles/6250](https://studygolang.com/articles/6250)

    浅显介绍了线程, 线程池, 协程, Actor, 等逐步发展起来的并发解决方案.
- [http://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html](http://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)

    Loom 是 JDK 上的“协程”方案, 现在应该还在开发中, 将来是否会真正地作为 JDK 的官方特性亦是未知数.

- [https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#project-loom](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#project-loom)

    最新版本的 RxJava3 说明文档中也提到了 project-Loom.

- [https://developer.android.com/kotlin/coroutines?hl=zh-cn](https://developer.android.com/kotlin/coroutines?hl=zh-cn)

    谷歌官方的 Kotlin 协程推介指南, 介绍了 Coroutine 的基本实现和一些使用细节. 文章标题虽然题名为“提升性能”, 但是其中涉及到 Coroutine 性能的地方也就对 WithContext 切换线程的相关表述. 认为这种提升应该也较小，不至于有质上的提升.

- [https://juejin.im/post/5e17f1135188254c7f57242e](https://juejin.im/post/5e17f1135188254c7f57242e)
    
    该博客的目的也是为了说明 Kotlin 协程的本质以及它与 Java 线程相比的异同, 驳斥了网络上许多不求甚解之人对 Coroutine 的错误认识.

- [https://piotrminkowski.com/2020/06/23/kotlin-coroutines-vs-java-threads/amp](https://piotrminkowski.com/2020/06/23/kotlin-coroutines-vs-java-threads/amp)

    亦是一篇剖析 Coroutine 与 Java#Thread 区别的文章, 主要通过几个例子说明了二者的以下几个区别: Kotlin 协程转到后台执行本质上依赖的依旧是线程池机制, 作为一个可以异步执行的代码块, 它在启动的时候并不需要指定到某个固定的线程上, 协程本身可以在一条线程被挂起, 在另一条线程上被恢复. 其实和线程池机制是十分雷同的, 只是协程这一进一步的抽象封装, 可以在异步执行完成后, 自动恢复到原来的执行位置, 让其继续执行下去. Java 线程是接受操作系统调度的, Java 线程一旦被阻塞, 便无法被继续使用, 直到阻塞并且再次被操作系统分配到时间片.

- [https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)

    对 Coroutine 的设计及实现原理进行了深入浅出的介绍.
- [the-difference-between-concurrency-as-a-library-and-concurrency-via-runtime-scheduler-distinction](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function)
    
    深入浅出地介绍了各种异步编程范式, 是一篇极好的文章, 目前只是粗略浏览, 还未深读.