# 使用 Kotlin 协程实现一些经典的 Java 多线程并发面试题

在 Java 面试中，为了考察多线程和并发相关的基础知识，常常会碰到如下几道典型的面试题：

- 两条线程交替运行，分别打印 0 和 1
- 三条线程按照严格的顺序执行，操作同一元素
- 五条线程线先一起打印 Hello，再一起打印 World
- 实现一个生产者消费者模式
- 实现一个死锁

本文尝试使用Kotlin Coroutine 来解答上述题目，以练习 Kotlin Coroutine 的使用。

## 两个协程交替执行

题目描述：启动两个协程，CoroutineA 和 CoroutineB，他们会交替打印字母 A 和字母 B。

```kotlin
fun main() = runBlocking {
    twoCoroutineCommunicate()
}

suspend fun twoCoroutineCommunicate() {
    // channel for communication
    val channel = Channel<Unit>()

    // repeat counts
    val counts = 10

    coroutineScope {
        val coroutineA = launch {
            repeat(counts) {
                delay(500)
                println("coroutineA on ${Thread.currentThread()}: A")
                // send signal to coroutineB
                channel.send(Unit)
                // suspend to wait for coroutineB's signal
                channel.receive()
            }
        }

        val coroutineB = launch {
            repeat(counts) {
                // once receive a signal from CoroutineA, print B
                channel.receive()
                delay(500)
                println("coroutineB on ${Thread.currentThread()}: B")
                // send signal to Coroutine A
                channel.send(Unit)
            }
        }
    }
    channel.close()
}
```

Kotlin 提供了类似于 GoLang 中的 Channel，协程可以使用 Channel 来进行通信。

上述代码中 CoroutineA 和 CoroutineB 通过一个 Channel 来互相发送信号，具体来说

- CoroutineA 打印字母 A 后调用```channel.send()```发送一个空消息给该 channel 的接受者(在该例中为 CoroutineB)
- CoroutineA 紧接着调用```channel.reveive()```挂起, 若是接收到 CoroutineB 通过该 channel 发送的消息后，CoroutineA 会继续执行进入下一次循环
- CoroutineB 一启动便会通过```channel.reveive()```挂起, 等待 channel 发来的消息, 一旦接收到 channel 发送来的消息便会执行其任务：
  - 打印字母 B
  - 发送消息给 CoroutineA

## 三个协程依序执行

题目描述：启动三个协程 CoroutineA, CoroutineB, CoroutineC, 他们会按照严格的先后次序去操作同一个元素。

```kotlin
fun main() = runBlocking {
    threeCoroutineCommunication()
}

suspend fun operateElement(
    name: String,
    element: MutableList<Int>,
    incomingChannel: Channel<Unit>,
    outgoingChannel: Channel<Unit>
) {
    repeat(10) {
        incomingChannel.receive()
        delay(200)
        element[0] = element[0] + 1
        println("$name: element: ${element[0]}")
        outgoingChannel.send(Unit)
    }
}

suspend fun threeCoroutineCommunication() {

    val job = Job()
    val scope = CoroutineScope(Dispatchers.Default + job)

    // a mutable common shared element
    val counter = mutableListOf(0)

    val channelA = Channel<Unit>()
    val channelB = Channel<Unit>()
    val channelC = Channel<Unit>()

    val coroutineA = scope.launch {
        operateElement("CoroutineA", counter, channelA, channelB)
    }

    val coroutineB = scope.launch {
        operateElement("CoroutineB", counter, channelB, channelC)
    }

    val coroutineC = scope.launch {
        operateElement("CoroutineC", counter, channelC, channelA)
    }

    // active coroutineA to start
    channelA.send(Unit)

    // delay to  wait all coroutine finished
    delay(10000)

    // cancel all coroutine
    job.cancel()
    
    println("threeCoroutineCommunication#end")
}
```

同样地，依旧是使用 Channel 作为协调三个协程工作的工具。

在上述实现中，每个协程的任务被统一抽象为 ```operateElement()```函数，开始执行每次操作前都通过```incomingChannel.receive()```被挂起，当次操作结束后，使用```outgoingChannel.send(Unit)```唤醒挂起在outgoingChannel上的协程。

同时使用 channelA, channelB, channelC 三个 Channel，使 CoroutineA, CoroutineB, CoroutineC 之间形成了一个“环状通信的关系”，使用任何一个 Channel 发送一个消息都可以唤醒挂起在其他 Channel 上的协程，形成三个协程依次执行的效果。

## 五个协程协调执行

题目描述：启动五个协程，它们先一起打印“Hello”，再一起打印"World"

```kotlin
fun main() = runBlocking {
    // 启动五个协程
    val jobs = List(5) {
        launch {
            // 先打印 "Hello"
            println("Hello")
            // 等待其他协程也完成 "Hello" 的打印
            yield()
            // 再打印 "World"
            println("World")
        }
    }
    // 等待所有协程完成
    jobs.forEach { it.join() }
}
```

上述代码中，使用了Kotlin 协程库中的```yield()```，其作用是挂起当前协程，使调度器调度其他协程来执行。

Java 中也提供了语义类似的方法```Thread.yield()```，但由于线程的调度是又操作系统决定的，因此 Java 线程的 yield 方法的作用只是提示操作系统当前线程愿意让渡资源，无法保证当前线程会被操作系统挂起。

## 生产者消费者

```Channel``` 提供了一种在协程之间安全传递数据流的方式，类似于并发安全的队列（BlockingQueue）。一个或多个协程可以向 Channel 发送 (send) 数据，一个或多个协程可以从中接收 (receive) 数据。因此从功能上看，Channel 天然地就是一种生产者消费者模型。下面的代码使用 ```Channel``` 实现了一个生产者消费者模型。

```kotlin
fun main() = runBlocking {
    val capacity = 5
    val channel = Channel<Int>(
        // capacity of the producer
        capacity,
        // action on buffer overflow, BufferOverflow.SUSPEND is Channel's Default setting
        onBufferOverflow = BufferOverflow.SUSPEND
    )

    val productIndex = AtomicInteger(0)

    // launch producer Coroutine
    val producerJobs = List(2) { index ->
        launch {
            while (true) {
                // producer will suspend if channel is full
                channel.send(productIndex.get()).also {
                    println("Producer$index#produce: ${productIndex.andIncrement}")
                }
                delay(Random.nextLong(100, 300))
            }
        }
    }

    // launch Consumer Coroutine
    val consumerJobs = List(3) { index ->
        launch {
            // consumer will suspend if channel is empty
            for (item in channel) {
                println("Consumer$index#consume: $item")
                delay(Random.nextLong(200, 500))
            }
        }
    }

    delay(5_000)

    producerJobs.forEach { job ->
        job.cancel()
    }
    consumerJobs.forEach { job ->
        job.cancel()
    }
    
    channel.close()

    println("end")
}
```

## 死锁

[这篇文章](https://mp.weixin.qq.com/s/sEF4hHp7mBpGJpYXjMMhBA)中介绍了一种 Kotlin 协程导致死锁的情形，是一种隐蔽的、容易被忽略的情形。

具体来说，Dispatch.IO 和 Dispatchers.Default 在实现上使用了同一个 Java 线程池，再加上 runBlocking 本质上会阻塞当前线程，因此，嵌套使用 runBlocking 并指定 Dispatchers.IO 或 Dispatchers.Default 时，容易导致线程池资源耗尽，进而引发死锁。
