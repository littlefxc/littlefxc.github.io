---
title: 线程池ThreadPoolExecutor
date: 2021-08-04 22:01:38
tags:
- Java
- 线程池
- 多线程
---

# 前言

线程就是一个放线程的池子。

使用线程池的好处：

- 重用已存在的线程，从而减少对象创建和销毁的开销。
- 控制并发，从而提高资源利用率，有效避免过多的资源竞争，提升性能
- 功能强大，有定时执行、定期执行、单线程执行、并发控制等等

<!-- more -->

从某种意义上讲，线程池是**特殊的对象池**。

这篇文章[commons-pool对象池（2）---实现一个线程池](https://blog.csdn.net/hit100410628/article/details/72934353)就介绍了怎么用 commons-pool2 实现线程池。

# 线程池的创建

我们可以通过 `ThreadPoolExecutor` 来创建一个线程池。

```java
public ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler)
```

构造方法中的字段含义如下：

- **corePoolSize**：核心线程数量，当有新任务在execute()方法提交时，会执行以下判断：

  1. 如果运行的线程少于 corePoolSize，则创建新线程来处理任务，即使线程池中的其他线程是空闲的；
  2. 如果线程池中的线程数量大于等于 corePoolSize 且小于 maximumPoolSize，则只有当workQueue满时才创建新的线程去处理任务；
  3. 如果设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池的大小是固定的，这时如果有新任务提交，若workQueue未满，则将请求放入workQueue中，等待有空闲的线程去从workQueue中取任务并处理；
  4. 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue已经满了，则通过handler所指定的策略来处理任务；

  所以，任务提交时，判断的顺序为 corePoolSize --> workQueue --> maximumPoolSize。

- **maximumPoolSize**：最大线程数量；

- **workQueue**：用于保存等待执行的任务的阻塞队列。当任务提交时，如果线程池中的线程数量大于等于corePoolSize的时候，把该任务封装成一个Worker对象放入队列中。可以选择以下几个阻塞队列。

  1. [ArrayBlockingQueue](https://www.cnblogs.com/leesf456/p/5533770.html)：是一个基于数组结构的**有界**阻塞队列，此队列按 FIFO 原则对元素进行排序。
  2. [LinkedBlockingQueue](https://blog.csdn.net/sinat_36553913/article/details/79533606)：一个基于链表结构的**无界**阻塞队列，此队列按 FIFO 排序元素，吞吐量通常高于 ArrayBlockingQueue。静态工厂方法 `Executors.newFixedThreadPool()` 使用了这个队列。
  3. [SynchronousQueue](https://juejin.im/post/5ae754c7f265da0ba76f8534)：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于 LinkedBlockingQueue，静态工厂方法 `Executors.newCachedThreadPool()` 使用了这个队列。
  4. [PriorityBlockingQueue](https://www.cnblogs.com/duanxz/archive/2012/10/22/2733947.html)：一个具有优先级的无限阻塞队列。
  5. [LinkedTransferQueue](https://juejin.im/post/5ae7561b6fb9a07aab29a2b2)：是一个基于链表结构的无界阻塞队列，是 SynchronousQueue 和 LinkedBlockingQueue 的合体，性能比 LinkedBlockingQueue 更高（没有锁操作），比 SynchronousQueue能存储更多的元素。

- **keepAliveTime**：线程池维护线程所允许的空闲时间。当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了keepAliveTime。所以如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。

  > PS: executor.allowCoreThreadTimeOut(true)// 核心线程超过空闲时间也会被回收

- **threadFactory**：它是 ThreadFactory 类型的变量，用来创建新线程。默认使用`Executors.defaultThreadFactory()` 来创建线程。使用默认的 ThreadFactory 来创建线程时，会使新创建的线程具有相同的`Thread.NORM_PRIORITY`优先级并且是非守护线程，同时也设置了线程的名称。使用开源框架 guava 提供的 ThreadFactoryBuilder 可以快速给线程池里的线程设置有意义的名字

  ```java
  new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
  ```

- **handler**：它是 RejectedExecutionHandler 类型的变量，表示线程池的**饱和策略**。如果队列和线程池都满了，线程池处于饱和状态，这时如果继续提交任务，就需要采取一种策略处理该任务。线程池提供了4种策略：

  1. AbortPolicy：直接抛出异常，这是默认策略；
  2. CallerRunsPolicy：用调用者所在的线程来执行任务；
  3. DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
  4. DiscardPolicy：直接丢弃任务；

  当然也可以根据应用场景需要自定义实现 RejectedExecutionHandler 接口自定义策略。

# 核心API - 操作类

- execute() : 提交任务，交给线程池运行
- submit(): 提交任务，能够返回结果
- shutdown():关闭线程池，等待任务都执行
- shutdownNow():关闭线程池，不等任务执行完（很少使用）

# 核心API- 监控类

通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用以下方法

- **getTaskCount**：线程池已经执行的和未执行的任务总数；
- **getCompletedTaskCount**：线程池已完成的任务数量，该值小于等于taskCount；
- **getLargestPoolSize**：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize；
- **getPoolSize**：线程池当前的线程数量；
- **getActiveCount**：当前线程池中正在执行任务的线程数量。

通过这些方法，可以对线程池进行监控，在ThreadPoolExecutor类中提供了几个空方法，如beforeExecute方法，afterExecute方法和terminated方法，可以扩展这些方法在执行前或执行后增加一些新的操作，例如统计线程池的执行任务的时间等，可以继承自ThreadPoolExecutor来进行扩展。这几个方法在线程池里是空方法。

```java
protected void beforeExecute(Thread t, Runnable r) { }
```

# 线程池状态

![image-20210804224722984](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210804224722984.png)

# 合理的配置线程池

任务一般可分为：CPU密集型、IO密集型、混合型，对于不同类型的任务需要分配不同大小的线程池。

- CPU密集型任务 尽量使用较小的线程池，一般为CPU核心数+1。 因为CPU密集型任务使得CPU使用率很高，若开过多的线程数，只能增加上下文切换的次数，因此会带来额外的开销。

- IO密集型任务 可以使用稍大的线程池，一般为2*CPU核心数。 IO密集型任务CPU使用率并不高，因此可以让CPU在等待IO的时候去处理别的任务，充分利用CPU时间。

- 混合型任务 可以将任务分成IO密集型和CPU密集型任务，然后分别用不同的线程池去处理。 只要分完之后两个任务的执行时间相差不大，那么就会比串行执行来的高效。 因为如果划分之后两个任务执行时间相差甚远，那么先执行完的任务就要等后执行完的任务，最终的时间仍然取决于后执行完的任务，而且还要加上任务拆分与合并的开销，得不偿失。

  估算的经验公式：

  ```
  N * U * (1 + WT/ST)
  
  N: CPU 核心数
  U: 目标 CPU 利用率
  WT: 线程等待时间
  ST: 线程运行时间
  ```

  ## 工具类示例(详细说明见[threading-stories-about-robust-thread](https://www.javacodegeeks.com/2012/03/threading-stories-about-robust-thread.html))：

  主要是继承下面这个抽象类实现方法就可以了

  ```java
  /**
   * A class that calculates the optimal thread pool boundaries. It takes the desired target utilization and the desired
   * work queue memory consumption as input and retuns thread count and work queue capacity.
   *
   * @author Niklas Schlimm
   */
  public abstract class PoolSizeCalculator {
  
      /**
       * The sample queue size to calculate the size of a single {@link Runnable} element.
       */
      private final int SAMPLE_QUEUE_SIZE = 1000;
  
      /**
       * Accuracy of test run. It must finish within 20ms of the testTime otherwise we retry the test. This could be
       * configurable.
       */
      private final int EPSYLON = 20;
      /**
       * Time (millis) of the test run in the CPU time calculation.
       */
      private final long testtime = 3000;
      /**
       * Control variable for the CPU time investigation.
       */
      private volatile boolean expired;
  
      /**
       * Calculates the boundaries of a thread pool for a given {@link Runnable}.
       *
       * @param targetUtilization    the desired utilization of the CPUs (0 <= targetUtilization <= 1)
       * @param targetQueueSizeBytes the desired maximum work queue size of the thread pool (bytes)
       */
      protected void calculateBoundaries(BigDecimal targetUtilization, BigDecimal targetQueueSizeBytes) {
          calculateOptimalCapacity(targetQueueSizeBytes);
          Runnable task = creatTask();
          start(task);
          start(task); // warm up phase
          long cputime = getCurrentThreadCPUTime();
          start(task); // test intervall
          cputime = getCurrentThreadCPUTime() - cputime;
          long waittime = (testtime * 1000000) - cputime;
          calculateOptimalThreadCount(cputime, waittime, targetUtilization);
      }
  
      private void calculateOptimalCapacity(BigDecimal targetQueueSizeBytes) {
          long mem = calculateMemoryUsage();
          BigDecimal queueCapacity = targetQueueSizeBytes.divide(new BigDecimal(mem), RoundingMode.HALF_UP);
          System.out.println("Target queue memory usage (bytes): " + targetQueueSizeBytes);
          System.out.println("createTask() produced " + creatTask().getClass().getName() + " which took " + mem
                  + " bytes in a queue");
          System.out.println("Formula: " + targetQueueSizeBytes + " / " + mem);
          System.out.println("* Recommended queue capacity (bytes): " + queueCapacity);
      }
  
      /**
       * Brian Goetz' optimal thread count formula, see 'Java Concurrency in Practice' (chapter 8.2)
       *
       * @param cpu               cpu time consumed by considered task
       * @param wait              wait time of considered task
       * @param targetUtilization target utilization of the system
       */
      private void calculateOptimalThreadCount(long cpu, long wait, BigDecimal targetUtilization) {
          BigDecimal waitTime = new BigDecimal(wait);
          BigDecimal computeTime = new BigDecimal(cpu);
          BigDecimal numberOfCPU = new BigDecimal(Runtime.getRuntime().availableProcessors());
          BigDecimal optimalthreadcount = numberOfCPU.multiply(targetUtilization).multiply(
                  new BigDecimal(1).add(waitTime.divide(computeTime, RoundingMode.HALF_UP)));
          System.out.println("Number of CPU: " + numberOfCPU);
          System.out.println("Target utilization: " + targetUtilization);
          System.out.println("Elapsed time (nanos): " + (testtime * 1000000));
          System.out.println("Compute time (nanos): " + cpu);
          System.out.println("Wait time (nanos): " + wait);
          System.out.println("Formula: " + numberOfCPU + " * " + targetUtilization + " * (1 + " + waitTime + " / "
                  + computeTime + ")");
          System.out.println("* Optimal thread count: " + optimalthreadcount);
      }
  
      /**
       * Runs the {@link Runnable} over a period defined in {@link #testtime}. Based on Heinz Kabbutz' ideas
       * (http://www.javaspecialists.eu/archive/Issue124.html).
       *
       * @param task the runnable under investigation
       */
      public void start(Runnable task) {
          long start = 0;
          int runs = 0;
          do {
              if (++runs > 5) {
                  throw new IllegalStateException("Test not accurate");
              }
              expired = false;
              start = System.currentTimeMillis();
              Timer timer = new Timer();
              timer.schedule(new TimerTask() {
                  public void run() {
                      expired = true;
                  }
              }, testtime);
              while (!expired) {
                  task.run();
              }
              start = System.currentTimeMillis() - start;
              timer.cancel();
          } while (Math.abs(start - testtime) > EPSYLON);
          collectGarbage(3);
      }
  
      private void collectGarbage(int times) {
          for (int i = 0; i < times; i++) {
              System.gc();
              try {
                  Thread.sleep(10);
              } catch (InterruptedException e) {
                  Thread.currentThread().interrupt();
                  break;
              }
          }
      }
  
      /**
       * Calculates the memory usage of a single element in a work queue. Based on Heinz Kabbutz' ideas
       * (http://www.javaspecialists.eu/archive/Issue029.html).
       *
       * @return memory usage of a single {@link Runnable} element in the thread pools work queue
       */
      public long calculateMemoryUsage() {
          BlockingQueue<Runnable> queue = createWorkQueue();
          for (int i = 0; i < SAMPLE_QUEUE_SIZE; i++) {
              queue.add(creatTask());
          }
          long mem0 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
          long mem1 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
          queue = null;
          collectGarbage(15);
          mem0 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
          queue = createWorkQueue();
          for (int i = 0; i < SAMPLE_QUEUE_SIZE; i++) {
              queue.add(creatTask());
          }
          collectGarbage(15);
          mem1 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
          return (mem1 - mem0) / SAMPLE_QUEUE_SIZE;
      }
  
      /**
       * Create your runnable task here.
       *
       * @return an instance of your runnable task under investigation
       */
      protected abstract Runnable creatTask();
  
      /**
       * Return an instance of the queue used in the thread pool.
       *
       * @return queue instance
       */
      protected abstract BlockingQueue<Runnable> createWorkQueue();
  
      /**
       * Calculate current cpu time. Various frameworks may be used here, depending on the operating system in use. (e.g.
       * http://www.hyperic.com/products/sigar). The more accurate the CPU time measurement, the more accurate the results
       * for thread count boundaries.
       *
       * @return current cpu time of current thread
       */
      protected abstract long getCurrentThreadCPUTime();
  
  }
  ```
  
  下面是示例：

  ```java
  /**
   * 如何合理设置线程池的大小
   */
  public class MyPoolSizeCalculator extends PoolSizeCalculator {
  
      public static void main(String[] args) {
          MyPoolSizeCalculator calculator = new MyPoolSizeCalculator();
          calculator.calculateBoundaries(
                  // CPU目标利用率
                  new BigDecimal(1.0),
                  // blockingqueue占用的内存大小，byte
                  new BigDecimal(100000));
  
          ThreadPoolExecutor executor =
                  new ThreadPoolExecutor(
                          8,
                          8,
                          // 默认情况下指的是非核心线程的空闲时间
                          // 如果allowCoreThreadTimeOut=true：核心线程/非核心线程允许的空闲时间
                          10L,
                          TimeUnit.SECONDS,
                          new LinkedBlockingQueue<>(2500),
                          Executors.defaultThreadFactory(),
                          new ThreadPoolExecutor.AbortPolicy()
                  );
      }
  
      @Override
      protected long getCurrentThreadCPUTime() {
          // 当前线程占用的总时间
          return ManagementFactory.getThreadMXBean().getCurrentThreadCpuTime();
      }
  
      @Override
      protected Runnable creatTask() {
          return new AsynchronousTask();
      }
  
      @Override
      protected BlockingQueue createWorkQueue() {
          return new LinkedBlockingQueue<>();
      }
  
  }
  
  class AsynchronousTask implements Runnable {
  
      @Override
      public void run() {
          // System.out.println(Thread.currentThread().getName());
      }
  }
  ```

  

# BlockQueue详解、选择与调优

- BlockQueue该怎么使用？
- 不同的BlockQueue对线程池的影响是什么？
- 几点线程池的调优技巧

## BlockQueue是什么

- BlockQueue 是阻塞队列
- 当队列为空时，获取对象会阻塞；当队列满时，放入对象会阻塞。

## BlockQueue的作用

- 实现队列的基本功能
- 多线程环境下自动管理线程的等待与唤醒

## BlockQueue 核心API

![image-20210804230153040](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210804230153040.png)

## 常用BlockQueue

![image-20210804230529147](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210804230529147.png)

## 调优技巧

- 合理设置corePoolSize、maximumPoolSize、workQueue的容量

比如我门想降低系统资源的消耗，

例如CPU的使用率，操作系统资源的消耗，上下文切换的开销，那么我门可以设置比较大的队列容量和一个比较小的线程池容量，例如：corePoolSize=5，maximumPoolSize=10，workQueue=LinkedBlockQueue(100)。

假如任务经常发生阻塞，任务队列经常满，那么可以重设maximumPoolSize。

# Executors 工具API

![image-20210805230140225](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210805230140225.png)

