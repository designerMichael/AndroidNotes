---
title: Java多线程
date: 2016-06-11 15:27:29
categories: Java
tags: 多线程
---



# 一些概念

使用多线程的目的：更好的利用cpu的资源

<!--more-->

线程:是操作系统能够进行运算调度的最小单位，它被包含在进程之中，是进程中的实际运作单位

进程：线程是进程的子集，一个进程可以有很多线程，每条线程并行执行不同的任务。不同的进程使用不同的内存空间，而所有的线程共享一片相同的内存空间。


多线程：指的是这个程序（一个进程）运行时产生了不止一个线程

线程池：基本思想还是一种对象池的思想，开辟一块内存空间，里面存放了众多(未死亡)的线程，池中线程执行调度由池管理器来处理。当有线程任务时，从池中取一个，执行完成后线程对象归池，这样可以避免反复创建线程对象所带来的性能开销，节省了系统的资源。

并行：多个cpu实例或者多台机器同时执行一段处理逻辑，是真正的同时。

并发：通过cpu调度算法，让用户看上去同时执行，实际上从cpu操作层面不是真正的同时。

线程安全：用来描绘一段代码。指在并发的情况之下，该代码经过多线程使用，线程的调度顺序不影响任何结果。反过来，线程不安全就意味着线程的调度顺序会影响最终结果，如不加事务的转账代码：
```java
void transferMoney(User from, User to, float amount){
  to.setMoney(to.getBalance() + amount);
  from.setMoney(from.getBalance() - amount);
}
```

同步：Java中的同步指的是通过人为的控制和调度，保证共享资源的多线程访问成为线程安全，来保证结果的准确。如上面的代码简单加入@synchronized关键字。线程安全的优先级高于性能。

# 线程调度策略


(1) 抢占式调度策略

Java运行时系统的线程调度算法是抢占式的。Java运行时系统支持一种简单的固定优先级的调度算法。如果一个优先级比其他任何处于可运行状态的线程都高的线程进入就绪状态，那么运行时系统就会选择该线程运行。新的优先级较高的线程抢占了其他线程。但是Java运行时系统并不抢占同优先级的线程。换句话说，Java运行时系统不是分时的。然而，基于Java Thread类的实现系统可能是支持分时的，因此编写代码时不要依赖分时。当系统中的处于就绪状态的线程都具有相同优先级时，线程调度程序采用一种简单的、非抢占式的轮转的调度顺序。

(2) 时间片轮转调度策略

有些系统的线程调度采用时间片轮转调度策略。这种调度策略是从所有处于就绪状态的线程中选择优先级最高的线程分配一定的CPU时间运行。该时间过后再选择其他线程运行。只有当线程运行结束、放弃(yield)CPU或由于某种原因进入阻塞状态，低优先级的线程才有机会执行。如果有两个优先级相同的线程都在等待CPU，则调度程序以轮转的方式选择运行的线程。


# 线程状态

![](http://upload-images.jianshu.io/upload_images/1689841-383f7101e6588094.png?imageMogr2/auto-orient/strip%7CimageView2/2)

除去起始（new）状态和结束（Dead）状态，线程有三种状态，分别是：就绪（Runnable）、运行（running）和阻塞（blocked）。其中就绪状态代表线程具备了运行的所有条件，只等待 CPU 调度（万事俱备，只欠东风）；处于运行状态的线程可能因为 CPU 调度（时间片用完了）的原因回到就绪状态，也有可能因为调用了线程的 yield 方法回到就绪状态，此时线程不会释放它占有的资源的锁，坐等 CPU 以继续执行；运行状态的线程可能因为 I/O 中断、线程休眠、调用了对象的 wait 方法而进入阻塞状态（有的地方也称之为等待状态）；而进入阻塞状态的线程会因为休眠结束、调用了对象的 notify 方法或 notifyAll 方法或其他线程执行结束而进入就绪状态。注意：调用 wait 方法会让线程进入等待池中等待被唤醒， notify 方法或 notifyAll 方法会让等待锁中的线程从等待池进入等锁池，在没有得到对象的锁之前，线程仍然无法获得 CPU 的调度和执行。


值得一提的是"blocked"这个状态：线程在Running的过程中可能会遇到阻塞(Blocked)情况

- 调用join()和sleep()方法，sleep()时间结束或被打断，join()中断,IO完成都会回到Runnable状态，等待JVM的调度。
- 调用wait()，使该线程处于等待池(wait blocked pool),直到notify()/notifyAll()，线程被唤醒被放到锁定池(lock blocked pool )，释放同步锁使线程回到可运行状态（Runnable）
- 对Running状态的线程加同步锁(Synchronized)使其进入(lock blocked pool ),同步锁被释放进入可运行状态(Runnable)。
- 在runnable状态的线程是处于被调度的线程，此时的调度顺序是不一定的。Thread类中的yield方法可以让一个running状态的线程转入runnable。

# 方法介绍


- sleep():使一个正在运行的线程处于睡眠状态，直到用 interrupt 方法来打断他的休眠或者 sleep 的休眠的时间到。是一个静态方法，调用此方法要捕捉InterruptedException 异常。


- wait():使一个线程处于等待（阻塞）状态(并且释放所持有的对象的锁）直到它被其他线程通过notify()或者notifyAll唤醒，该方法只能在同步方法中调用。

- notify():唤醒一个处于等待状态的线程，当然在调用此方法的时候，并不能确切的唤醒某一个等待状态的线程，而是由JVM确定唤醒哪个线程，而且与优先级无关；该方法只能在同步方法或同步块内部调用。


- notityAll():唤醒所有处入等待状态的线程，注意并不是给所有唤醒线程一个对象的锁，而是让它们竞争；同样该方法只能在同步方法或同步块内部调用。


wait()、notify()、notifyAll()这三个方法是 java.lang.Object 的 final native 方法，任何继承 java.lang.Object 的类都有这三个方法。调用这三个方法中任意一个，当前线程必须是锁的持有者，如果不是会抛出一个 IllegalMonitorStateException 异常。


- sleep() 和 wait() 有什么区别?  
答：sleep()方法是线程类（Thread）的静态方法，导致此线程暂停执行指定时间，将执行机会给其他线程，但是监控状态依然保持，到时后会自动恢复（线程回到就绪（ready）状态），因为调用 sleep 不会释放对象锁。wait() 是 Object 类的方法，对此对象调用 wait()方法导致本线程放弃对象锁(线程暂停执行)，进入等待此对象的等待锁定池，只有针对此对象发出 notify 方法（或 notifyAll）后本线程才进入对象锁定池准备获得对象锁进入就绪状态。


- sleep() 和 yield() 有什么区别?    
	- ① sleep() 方法给其他线程运行机会时不考虑线程的优先级，因此会给低优先级的线程以运行的机会；yield() 方法只会给相同优先级或更高优先级的线程以运行的机会；
	- ② 线程执行 sleep() 方法后转入阻塞（blocked）状态，而执行 yield() 方法后转入就绪（ready）状态；
	- ③ sleep() 方法声明抛出InterruptedException，而 yield() 方法没有声明任何异常；
	- ④ sleep() 方法比 yield() 方法（跟操作系统相关）具有更好的可移植性。


# 同步工具
synchronized, wait, notify 是任何对象都具有的同步工具。

首先要明确monitor的概念，Java中的每个对象都有一个监视器，来监测并发代码的重入。在非多线程编码时该监视器不发挥作用，反之如果在synchronized 范围内，监视器发挥作用。

## synchronized用法

### 代码块
在多线程环境下，synchronized块中的方法获取了lock实例的monitor，如果实例相同，那么只有一个线程能执行该块内容
```java
public class Thread1 implements Runnable {
   Object lock;
   public void run() {  
       synchronized(lock){
         ..do something
       }
   }
}
```

### 直接用于方法
相当于上面代码中用lock来锁定的效果，实际获取的是Thread1类的monitor。更进一步，如果修饰的是static方法，则锁定该类所有实例。
```java
public class Thread1 implements Runnable {
   public synchronized void run() {  
        ..do something
   }
}
```

## synchronized, wait, notify结合:典型场景生产者消费者问题

```java
  /**
   * 生产者生产出来的产品交给店员
   */
  public synchronized void produce()
  {
      if(this.product >= MAX_PRODUCT)
      {
          try
          {
              wait();  
              System.out.println("产品已满,请稍候再生产");
          }
          catch(InterruptedException e)
          {
              e.printStackTrace();
          }
          return;
      }

      this.product++;
      System.out.println("生产者生产第" + this.product + "个产品.");
      notifyAll();   //通知等待区的消费者可以取出产品了
  }

  /**
   * 消费者从店员取产品
   */
  public synchronized void consume()
  {
      if(this.product <= MIN_PRODUCT)
      {
          try 
          {
              wait(); 
              System.out.println("缺货,稍候再取");
          } 
          catch (InterruptedException e) 
          {
              e.printStackTrace();
          }
          return;
      }

      System.out.println("消费者取走了第" + this.product + "个产品.");
      this.product--;
      notifyAll();   //通知等待去的生产者可以生产产品了
  }
```

## volatile

线程为了提高效率，会将某成员变量(如A)拷贝了一份（如B）（到自己的线程栈中），线程中对A的访问其实访问的是B，最后才进行A和B的同步。因此中间时刻存在A和B不一致的情况。这个时候如果另外的线程抢到cpu资源而去访问这个变量A，就会得到修改前的值，而我们希望的是得到修改后的值。

volatile就是用来避免这种情况的。volatile告诉jvm， 它所修饰的变量不保留拷贝，直接访问主内存中的（也就是上面说的A) 。本质上，volatile就是不去缓存，直接取值。在线程安全的情况下加volatile会牺牲性能。

volatile是一个特殊的修饰符，只有成员变量才能使用它。


volatile和synchronized的区别：volatile是变量修饰符，而synchronized则作用于一段代码或方法



# 基本线程类


基本线程类指的是Thread类，Runnable接口，Callable接口

## Thread类

Thread 类实现了Runnable接口。

### 基本使用

创建并启动线程步骤：
1. 继承Thread类，覆盖run()方法。   
2. new出线程对象并用start()方法启动线程。
```java
class DemoThread extends Thread {

    @Override
    public void run() {
        super.run();
        // Perform time-consuming operation...
    }
}

DemoThread t = new DemoThread();
t.start();
```

### Thread类相关方法



#### join()
join() 方法定义在 Thread 类中，所以调用者必须是一个线程，join() 方法的意思是等待该线程终止，主要作用是让调用该方法的 Thread 完成 run() 方法里面的东西后，再执行 join() 方法后面的代码，看下下面的”意思”代码：
```java
Thread t1 = new Thread(计数线程一);  
Thread t2 = new Thread(计数线程二);  
t1.start();  
t1.join(); // 等待计数线程一执行完成，再执行计数线程二
t2.start();  
```
启动 t1 后，调用了 join() 方法，直到 t1 的计数任务结束，才轮到 t2 启动，然后 t2 才开始计数任务，两个线程是按着严格的顺序来执行的。如果 t2 的执行需要依赖于 t1 中的完整数据的时候，这种方法就可以很好的确保两个线程的同步性。


#### Thread.yield() 
线程放弃运行，将CPU的控制权让出。 yield() 方法让出控制权后，还有可能马上被系统的调度机制选中来运行，比如，执行yield()方法的线程优先级高于其他的线程，那么这个线程即使执行了 yield() 方法也可能不能起到让出CPU控制权的效果，因为它让出控制权后，进入排队队列，调度机制将从等待运行的线程队列中选出一个等级最高的线程来运行，那么它又（很可能）被选中来运行。

#### sleep()：睡眠

#### interrupte()：中断


## Runnable接口

基本使用：实现Runnable接口，重写run方法。再把实现了Runnable接口的对象放入Thread的构造中，再start。分两种，用匿名内部类和不用匿名内部类。推荐使用匿名内部类。

```java
class MyThread implements Runnable{
	public void run(){
		//具体逻辑
	}

}

MyThread t = new MyThread();
new Thread(t).start();

//用匿名内部类
new Thread(new Runnable(){
	public void run(){
		//具体逻辑
	}

}).start();
```


## Callable接口

Runnable 和 Callable 都代表那些要在不同的线程中执行的任务。Runnable 从 JDK1.0 开始就有了，Callable 是在 JDK1.5 增加的。它们的主要区别是 Callable 的 call() 方法可以返回值和抛出异常，而 Runnable 的 run() 方法没有这些功能。Callable 可以返回装载有计算结果的 Future 对象。

看一下 Runnable 和 Callable 的源码：
```java
public interface Runnable {
	public void run();
}

public interface Callable<V> {
	V call() throws Exception;
}
```

从源码中可以看出区别：

1）Callable 接口下的方法是 call()，Runnable 接口的方法是 run()。  
2）Callable 的任务执行后可返回值，而 Runnable 的任务是不能返回值的。  
3）call() 方法可以抛出异常，run()方法不可以的。  
4）运行 Callable 任务可以拿到一个 Future 对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过 Future 对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。  
5）Thread 类只支持 Runnable：(new Thread(Runnable r))，Callable不支持这种形式。不过 Callable 可以使用 ExecutorService


上面提到了 Future,看一下 Future 接口:

```java
public interface Future<V> {

	boolean cancel(boolean mayInterruptIfRunning);

	boolean isCancelled();

	boolean isDone();

	V get() throws InterruptedException, ExecutionException;

	V get(long timeout, TimeUnit unit)
    	throws InterruptedException, ExecutionException, TimeoutException;
}
```

Future 定义了5个方法：

1）boolean cancel(boolean mayInterruptIfRunning)：试图取消对此任务的执行。如果任务已完成、或已取消，或者由于某些其他原因而无法取消，则此尝试将失败。当调用 cancel() 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。如果任务已经启动，则 mayInterruptIfRunning 参数确定是否应该以试图停止任务的方式来中断执行此任务的线程。此方法返回后，对 isDone() 的后续调用将始终返回 true。如果此方法返回 true，则对 isCancelled() 的后续调用将始终返回 true。  

2）boolean isCancelled()：如果在任务正常完成前将其取消，则返回 true。

3）boolean isDone()：如果任务已完成，则返回 true。 可能由于正常终止、异常或取消而完成，在所有这些情况中，此方法都将返回 true。

4）V get()throws InterruptedException,ExecutionException：如有必要，等待计算完成，然后获取其结果。

5）V get(long timeout,TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException： 如有必要，最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）。

Future 接口也不能用在线程中，那怎么用，谁实现了 Future 接口呢？答：FutureTask。
```java
public class FutureTask<V> implements RunnableFuture<V> {
	...
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
	void run();
}
```
FutureTask 实现了 Runnable 和 Future，所以兼顾两者优点，既可以在 Thread 中使用，又可以在 ExecutorService 中使用。

具体使用的示例：
```java
Callable<String> callable = new Callable<String>() {
    @Override
    public String call() throws Exception {
        return "个人博客：sunfusheng.com";
    }
};

FutureTask<String> task = new FutureTask<String>(callable);

Thread t = new Thread(task);
t.start(); // 启动线程
task.cancel(true); // 取消线程
```


也可以和ExecutorService一起用：
```java
ExecutorService e = Executors.newFixedThreadPool(3);
 //submit方法有多重参数版本，及支持callable也能够支持runnable接口类型.
Future future = e.submit(new myCallable());
future.isDone() //return true,false 无阻塞
future.get() // return 返回值，阻塞直到该线程运行结束
```

使用 FutureTask 的好处是 FutureTask 是为了弥补 Thread 的不足而设计的，它可以让程序员准确地知道线程什么时候执行完成并获得到线程执行完成后返回的结果。FutureTask 是一种可以取消的异步的计算任务，它的计算是通过 Callable 实现的，它等价于可以携带结果的 Runnable，并且有三个状态：等待、运行和完成。完成包括所有计算以任意的方式结束，包括正常结束、取消和异常。


# 线程池


## 是什么

在面向对象编程中，创建和销毁对象是很费时间的，因为创建一个对象要获取内存资源或者其它更多资源。在 Java 中更是如此，虚拟机将试图跟踪每一个对象，以便能够在对象销毁后进行垃圾回收。所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数，特别是一些很耗资源的对象创建和销毁，这就是"池化资源"技术产生的原因。线程池顾名思义就是事先创建若干个可执行的线程放入一个池（容器）中，需要的时候从池中获取线程不用自行创建，使用完毕不需要销毁线程而是放回池中，从而减少创建和销毁线程对象的开销。

## 优点

1）避免线程的创建和销毁带来的性能开销。  
2）避免大量的线程间因互相抢占系统资源导致的阻塞现象。  
3｝能够对线程进行简单的管理并提供定时执行、间隔执行等功能。  


##相关类

Java里面线程池的顶级接口是 Executor，不过真正的线程池接口是 ExecutorService， ExecutorService 的默认实现是 ThreadPoolExecutor；普通类 Executors 里面调用的就是 ThreadPoolExecutor。

照例看一下各个接口的部分源码：
```java
public interface Executor {
	void execute(Runnable command);
}

public interface ExecutorService extends Executor {
	void shutdown();
	List<Runnable> shutdownNow();
	
	boolean isShutdown();
	boolean isTerminated();
	
	<T> Future<T> submit(Callable<T> task);
	<T> Future<T> submit(Runnable task, T result);
	Future<?> submit(Runnable task);
	...
}

public class Executors {
	public static ExecutorService newCachedThreadPool() {
    		return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, 
    						new SynchronousQueue<Runnable>());
	}
	...
}
```

通过 ThreadPoolExecutor 的构造函数，撸一撸线程池相关参数的概念：
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, 
    	threadFactory, defaultHandler);
}
```

1）corePoolSize：线程池的核心线程数，一般情况下不管有没有任务都会一直在线程池中一直存活，只有在 ThreadPoolExecutor 中的方法 allowCoreThreadTimeOut(boolean value) 设置为 true 时，闲置的核心线程会存在超时机制，如果在指定时间没有新任务来时，核心线程也会被终止，而这个时间间隔由第3个属性 keepAliveTime 指定。

2）maximumPoolSize：线程池所能容纳的最大线程数，当活动的线程数达到这个值后，后续的新任务将会被阻塞。

3）keepAliveTime：控制线程闲置时的超时时长，超过则终止该线程。一般情况下用于非核心线程，只有在 ThreadPoolExecutor 中的方法 allowCoreThreadTimeOut(boolean value) 设置为 true时，也作用于核心线程。

4）unit：用于指定 keepAliveTime 参数的时间单位，TimeUnit 是个 enum 枚举类型，常用的有：TimeUnit.HOURS(小时)、TimeUnit.MINUTES(分钟)、TimeUnit.SECONDS(秒) 和 TimeUnit.MILLISECONDS(毫秒)等。

5）workQueue：线程池的任务队列，通过线程池的 execute(Runnable command) 方法会将任务 Runnable 存储在队列中。

6）threadFactory：线程工厂，它是一个接口，用来为线程池创建新线程的。

## 创建线程池

比如：ExecutorService pool = Executors.newCachedThreadPool();

Executors 提供四种线程池：

1）newCachedThreadPool 是一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们。对于执行很多短期异步任务的程序而言，这些线程池通常可提高程序性能。调用 execute() 将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。因此，长时间保持空闲的线程池不会使用任何资源。注意，可以使用 ThreadPoolExecutor 构造方法创建具有类似属性但细节不同（例如超时参数）的线程池。

2）newSingleThreadExecutor 创建是一个单线程池，也就是该线程池只有一个线程在工作，所有的任务是串行执行的，如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它，此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

3）newFixedThreadPool 创建固定大小的线程池，每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小，线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

4）newScheduledThreadPool 创建一个大小无限的线程池，此线程池支持定时以及周期性执行任务的需求。

## 线程池的关闭

ThreadPoolExecutor 提供了两个方法，用于线程池的关闭，分别是 shutdown() 和 shutdownNow()。

shutdown()：不会立即的终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务。

shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务。

## 实例

待补充。。。





# 高级多线程控制类


## ThreadLocal类


用处：保存线程的独立变量。对一个线程类（继承自Thread)
当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。常用于用户登录控制，如记录session信息。

实现：每个Thread都持有一个TreadLocalMap类型的变量（该类是一个轻量级的Map，功能与map一样，区别是桶里放的是entry而不是entry的链表。功能还是一个map。）以本身为key，以目标为value。
主要方法是get()和set(T a)，set之后在map里维护一个threadLocal -> a，get时将a返回。ThreadLocal是一个特殊的容器。

...后面还有但是还不了解，以后再来撸
<http://www.jianshu.com/p/40d4c7aebd66>




# 死锁
什么是死锁：

两个进程都在等待对方执行完毕才能继续往下执行的时候就发生了死锁。结果就是两个进程都陷入了无限的等待中。


产生死锁的四个必要条件：

- 1 互斥条件：一个资源每次只能被一个进程使用。
- 2 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
- 3 不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺。
- 4 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。

死锁的解决方法:

- a 撤消陷于死锁的全部进程；
- b 逐个撤消陷于死锁的进程，直到死锁不存在；
- c 从陷于死锁的进程中逐个强迫放弃所占用的资源，直至死锁消失。
- d 从另外一些进程那里强行剥夺足够数量的资源分配给死锁进程，以解除死锁状态

# 参考链接

<http://sunfusheng.com/android/java/%E7%BA%BF%E7%A8%8B/2016/04/08/thread-threadpool.html>