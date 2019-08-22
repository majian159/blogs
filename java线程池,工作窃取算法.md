# 前言
在上一篇《java线程池,阿里为什么不允许使用Executors?》中我们谈及了线程池，同时又发现一个现象，当最大线程数还没有满的时候耗时的任务全部堆积给了单个线程, 代码如下:
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        1, //corePoolSize
        100, //maximumPoolSize
        100, //keepAliveTime
        TimeUnit.SECONDS, //unit
        new LinkedBlockingDeque<>(100));//workQueue

for (int i = 0; i < 5; i++) {
    final int taskIndex = i;
    executor.execute(() -> {
        System.out.println(taskIndex);
        try {
            Thread.sleep(Long.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
}
// 输出: 0
```
下图很形象的说明了这个问题:
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g68a050bs6j20cs0ed41c.jpg)

那么有没有一种机制，在线程池中还有线程可以提供服务的时候帮忙分担一些已经被分配给某一个线程的耗时任务呢？  
答案当然是有的：**工作窃取算法**

# 工作窃取 (Work stealing)
> 这边大家先不要将这个跟java挂钩，因为这个属于算法，一种思想和套路，并不是特定语言特有的东西，所以不同的语言对应的实现也不尽一样，但核心思想一致。
> **这边会用“工作者”来代替线程的说法，如果在java中这个工作者就是线程。**

**工作窃取核心思想是，自己的活干完了去看看别人有没有没干完的活，如果有就拿过来帮他干。**  
大多数实现机制是：**为每个工作者程分配一个双端队列(本地队列)用于存放需要执行的任务，当自己的队列没有数据的时候从其它工作者队列中获得一个任务继续执行。**  

我们来看一张图，这张图是发生了工作窃取时的状态。
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g68fr2v4clj20fw06ydg9.jpg)
可以看到工作者B的本地队列中没有了需要执行的规则，它正尝试从工作者A的任务队列中偷取一个任务。
> 为什么说尝试？因为涉及到并行编程肯定涉及到并发安全的问题，有可能在偷取过程中工作者A提前抢占了这个任务，那么B的偷取就会失败。大多数实现会尽量避免发生这个问题，所以大多数情况下不会发生。

## 并发安全的问题是怎么避免的呢？
一般是自己的本地队列采取LIFO(后进先出)，偷取时采用FIFO(先进先出)，一个从头开始执行，一个从尾部开始执行，由于偷取的动作十分快速，会大量降低这种冲突，也是一种优化方式。

# Java中的工作窃取算法线程池
在Java 1.7新增了一个ForkJoinPool类，主要是实现了工作窃取算法的线程池，该类在1.8中被优化了，同时1.8在Executors类中还新增了两个newWorkStealingPool工厂方法。

> java7中的fork/join task 和 java8中的并行stream都是基于ForkJoinPool。
```java
// 使用当前处理器数, 相当于调用 newWorkStealingPool(Runtime.getRuntime().availableProcessors());
public static ExecutorService newWorkStealingPool();
public static ExecutorService newWorkStealingPool(int parallelism);
```
同时 ForkJoinPool 还在全局建立了一个公共的线程池
```java
ForkJoinPool.commonPool();
```
默认的并行度是当前JVM识别到的处理器数。这个值也是可以通过参数进行变更的，下面是可以通过JVM熟悉进行commonPool设置的参数。

> 前缀统一为: java.util.concurrent.ForkJoinPool.common.
> 比如 parallelism 就要写为 java.util.concurrent.ForkJoinPool.common.parallelism

|  参数   | 描述  | 默认值 |
|  ----  | ----  | ---- |
| parallelism  | 并行级别 | JVM识别到的处理器数 |
| threadFactory  | 线程工厂类名 | ForkJoinPool.DefaultForkJoinWorkerThreadFactory |
| exceptionHandler  | 错误处理程序 | null |
| maximumSpares  | 最大允许额外线程数 | 256 |

使用工作窃取算法的线程池来优化之前的代码
```java
ExecutorService executor = Executors.newWorkStealingPool(8);

for (int i = 0; i < 5; i++) {
    final int taskIndex = i;
    executor.execute(() -> {
        System.out.println(taskIndex);
        try {
            Thread.sleep(Long.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
}

// 无序输出 0~4
```

### 如果将Executors.newWorkStealingPool(8)改成ForkJoinPool.commonPool()会输出什么？
如果你能知道输出什么那么你对这个机制就算掌握了，会输出当前运行环境中处理器（cpu）数量的次数（如果核算大于5就只会输出5个结果）。

## newWorkStealingPool 和 ForkJoinPool.commonPool 该优先选择哪个？
这个没有最优解，推荐执行的小任务（零散的）使用commonPool，而有特定目的的则使用`newWorkStealingPool`或 `new ForkJoinPool`。

## 使用ForkJoinPool.commonPool 需要注意的问题
`commonPool`默认使用当前环境的处理器格式来当做并行程度，如果遇上堵塞形任务一样会遇到浪费算力的问题。  
这点在容器化时需要特别注意，因为容器化的cpu个数限制往往不会太大。  
这种时候可以通过设置默认的并行度或者使用`newWorkStealingPool`来手动指定并行度。

# 最后
## 为什么ForkJoinPool极少出现线程关键字？
现在许多语言淡化了线程这个概念，而golang中更是直接去掉了线程能力改为提供协程`goroutine`。  
目的还是线程是OS的资源，OS对程序内部运行其实并没有太了解，为了避免线程资源的浪费许多语言会自己管理线程。  
对于程序来说我们关心的主要还是任务的并行运行，并不关心是线程还是协程。  
下面是一些对应关系：
- CPU : 线程 (1:N)
- 线程 : 协程 (1:N)

> CPU由OS管理，OS提供线程给程序使用，程序利用线程提供协程能力给应用使用。

## ForkJoinPool一定更快吗？
不，大家都知道做的事情越多逻辑越复杂效率会越低。  
ForkJoinPool中的工作队列，工作窃取都是需要额外管理的，同时也对线程调度和GC带来了压力。
所以ForkJoinPool并不是万能药大家根据具体需要去使用。

后面可能会跟大家分享下 Spring 中的 `@Async`。