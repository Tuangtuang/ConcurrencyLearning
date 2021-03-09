# 线程

## 线程的创建

### 方法1：new Thread

```java
Thread t = new Thread(){
  public void run(){
    // 要执行的任务
  }
};
// 启动线程
t.start();

```

```java
Thread t = new Thread("name"){
	@Override
  public void run(){
    // 要执行的任务
    System.out.println("run");
  }
};
// 启动线程
t.start();
```

### 方法2：使用Runnable配合Thread

- 把线程和任务分开
  - Thread代表线程
  - Runnable代表线程要执行的任务

```java
Runnable runnable = new Runnable(){
	public void run(){
		// 要执行的任务
	}
};

// 创建线程对象
Thread t=new Thread(runnable);
// 启动线程
t.start();
```

```java
Runnable runnable = new Runnable(){
  @Override
	public void run(){
		// 要执行的任务
    System.out.println("run");
	}
};

// 创建线程对象
Thread t=new Thread(runnable,"name");
// 启动线程
t.start();
```

#### Lambda简化

```java
Runnable runnable = () -> {
		// 要执行的任务
    System.out.println("run");
};

// 创建线程对象
Thread t=new Thread(runnable,"name");
// 启动线程
t.start();
```



### 方法3：使用FutureTask配合Thred

```java
// 创建任务对象
FutureTask<Integer> task3 = new FutureTask<>(new Callable<Integer>(){
		log.debug("hello");
		return 100;
	}
);
// 参数1 是任务对象; 参数2 是线程名字，
new Thread(task3, "t3").start();
// 主线程阻塞，同步等待 task 执行完毕的结果 
Integer result = task3.get(); 
log.debug("结果是:{}", result);
```



### Thread和Runnable原理

- 方法一重写了thread的run方法
- 方法二执行了Runnable对象的run方法

## 查看Java线程

- jps查看所有Java进程
- jstack <PID> 查看某个进程PID中的线程状态
- jconsole图形化界面

## 线程运行的原理

### 栈帧图解

![image-20210223103221064](/Users/i531515/Library/Application Support/typora-user-images/image-20210223103221064.png)

- 每一个线程都拥有自己的栈，且互相独立

### 线程的上下文切换

- 因为以下原因CPU不再执行当前线程，转而执行其他线程
  - 线程的CPU时间片用完了
  - 垃圾回收
  - 有更高优先级的线程需要运行
  - 线程调用了自己的sleep, yield, wait, join, park, synchronized, lock等方法

- 当上下文切换时，需要保存当前线程的状态。
  - Java中通过PC计数器保存下一条JVM指令的执行地址，是线程私有的。
  - 状态包括PC计数器，虚拟机栈中每一个栈的信息。
  - 频繁的上下文切换会影响性能。

## 常见方法

### start与run

- 调用run

  ```java
  public static void main(String[] args) { Thread t1 = new Thread("t1") {
  	@Override
  	public void run() { 
      log.debug(Thread.currentThread().getName()); 
      FileReader.read(Constants.MP4_FULL_PATH);
  	} 
  };
  t1.run();
  log.debug("do other things ..."); }
  ```

  ```
  19:39:14 [main] c.TestStart - main
  19:39:14 [main] c.FileReader - read [1.mp4] start ...
  19:39:18 [main] c.FileReader - read [1.mp4] end ... cost: 4227 ms 19:39:18 [main] c.TestStart - do other things ...
  ```

  - 程序仍然在main线程运行，仍然是同步的。

- 调用start

  ```
  19:41:30 [main] c.TestStart - do other things ...
  19:41:30 [t1] c.TestStart - t1
  19:41:30 [t1] c.FileReader - read [1.mp4] start ...
  19:41:35 [t1] c.FileReader - read [1.mp4] end ... cost: 4542 ms
  ```

  - 异步
  - 程序只能start一次，如果多次start，会显示illegal thread state

- 直接调用 run 是在主线程中执行了 run，没有启动新的线程；使用 start 是启动新的线程，通过新的线程间接执行 run 中的代码

### sleep和yield

- sleep
  - 调用sleep会使得当前线程从running 进入 timed waiting（阻塞）状态
  - 其他线程可以打断正在睡眠的线程，这时sleep方法会抛出InterruptedException
  - 睡眠结束后线程不一定会马上得到执行
  - 建议使用Time Unit的sleep代替Thread的sleep来获得更好的可读性
- yield
  - 调用 yield 会让当前线程从 Running 进入 Runnable 就绪状态，然后调度执行其它线程
  - 具体的实现依赖于操作系统的任务调度器
    - 如果CPU中没有其他线程要运行，CPU仍然会执行当前线程

#### sleep的应用

- 防止程序对CPU占用100%，使cpu空转浪费资源

  - ```java
     while(true) { 
       try {
      			Thread.sleep(50);
      		} catch (InterruptedException e) {
      			e.printStackTrace(); 
       }
    }
    ```

- 可以用 wait 或 条件变量达到类似的效果，不同的是，后两种都需要加锁，并且需要相应的唤醒操作，一般适用于要进行同步的场景，sleep 适用于无需锁同步的场景。

### 优先级

- 线程优先级会提示(hint)调度器优先调度该线程，但它仅仅是一个提示，调度器可以忽略它
- 如果 cpu 比较忙，那么优先级高的线程会获得更多的时间片，但 cpu 闲时，优先级几乎没作用

### join

#### 无时效join

- 等待线程运行结束

- ```java
  static int r = 0;
  public static void main(String[] args) throws InterruptedException {
  	test1(); 
  }
  private static void test1() throws InterruptedException { 
  	log.debug("开始");
  	Thread t1 = new Thread(() -> {
  		log.debug("开始"); sleep(1); 
      log.debug("结束"); 
      r = 10;
  	});
  	t1.start(); 
    //t1.join();
  	log.debug("结果为:{}", r); 
  	log.debug("结束");
  }
  ```

- 输入结果为0

- 添加t1.join();之后，主线程会等待t1执行结束之后，再继续执行

- 应用：下面这个方法cost是多少秒？ 2 秒

  - ```java
    static int r1 = 0;
    static int r2 = 0;
    public static void main(String[] args) throws InterruptedException {
    	test2(); 
    }
    private static void test2() throws InterruptedException { 
      Thread t1 = new Thread(() -> {
    	sleep(1);
    	r1 = 10; 
      }
                            );
    	Thread t2 = new Thread(() -> { 
        sleep(2);
    		r2 = 20;
    	}
                            );
    	long start = System.currentTimeMillis();
    	t1.start();
    	t2.start();
    	t1.join();
    	t2.join();
    	long end = System.currentTimeMillis();
    	log.debug("r1: {} r2: {} cost: {}", r1, r2, end - start);
    }
    ```

  - 第一个 join:等待 t1 时, t2 并没有停止, 而在运行

    第二个 join:1s 后, 执行到此, t2 也运行了 1s, 因此也只需再等待 1s

#### 有时效join

- join(long n)
- 等待n毫秒，还没结束就不等了

### interrupt

- 打断处于wait，sleep，join状态（处于阻塞状态）的线程，会清除打断标记（将isInterrupted设置为假）

  - ```java
    private static void test1() throws InterruptedException { 
      Thread t1 = new Thread(()->{
    	sleep(1); 
      }, "t1" );
    	t1.start();
    	sleep(0.5);
    	t1.interrupt();
    	log.debug(" 打断状态: {}", t1.isInterrupted());
    }
    ```

  - ```
    java.lang.InterruptedException: sleep interrupted
    at java.lang.Thread.sleep(Native Method)
    at java.lang.Thread.sleep(Thread.java:340)
    at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386) at cn.itcast.n2.util.Sleeper.sleep(Sleeper.java:8)
    at cn.itcast.n4.TestInterrupt.lambda$test1$3(TestInterrupt.java:59)
    at java.lang.Thread.run(Thread.java:745) 21:18:10.374 [main] c.TestInterrupt - 打断状态: false
    ```

- 打断正在运行的线程，不清楚打断标记

  - ```java
    private static void test2() throws InterruptedException { 
      Thread t2 = new Thread(()->{
    		while(true) {
    			Thread current = Thread.currentThread(); 
        	//如果不加这一部分代码，打断不会对本线程有任何影响
          //主线程只是告诉本线程我要打断你
          //boolean interrupted = current.isInterrupted(); 
          //if(interrupted) {
    				//log.debug(" 打断状态: {}", interrupted);
    				//break; 
          //}
    	}
    	}, "t2");
    	t2.start();
    	sleep(0.5);
    	t2.interrupt(); 
    }
    ```

  - ```
    20:57:37.964 [t2] c.TestInterrupt - 打断状态: true
    ```

- 打断park线程
  
  - 打断 park 线程, 不会清空打断状态

#### **<font color="red">设计模式——两阶段终止</font>**

- 如何在T1线程中“优雅地”终止T2线程？“优雅”指给T2线程料理后事的机会

  - 错误思路

    - 用T2.stop()强制停止
      - 如果t2对共享资源加了锁，用T2.stop()无法释放共享资源的锁
    - 用System.exit()
      - 杀死了进程和进程中的所有线程

  - 利用isInterrupted

    - ![image-20210224133937977](/Users/i531515/Library/Application Support/typora-user-images/image-20210224133937977.png)

    - ```java
      class TPTInterrupt { 
        private Thread thread;
        
      	public void start(){
      		thread = new Thread(() -> {
      			while(true) {
      				Thread current = Thread.currentThread(); 
              if(current.isInterrupted()) {
      						log.debug("料理后事");
      						break; 
              }
      				try {
      					Thread.sleep(1000); //情况1:在本处被打断，会奖打断标记设置为假，会一直循环，无法正常打断
              	log.debug("将结果保存");//情况2:被打断，打断标记仍然为真，会料理后事
      				} catch (InterruptedException e) {
               // 重新设置打断标记，设置为真，进入while循环，会料理后事
                current.interrupt();
              }
      				// 执行监控操作 
            }
      		},"监控线程");
      	thread.start(); 
       }
        
      	public void stop() { 
        	thread.interrupt();
      	} 
      }
      
      //调用
      TPTInterrupt t = new TPTInterrupt(); 
      t.start();
      Thread.sleep(3500); 
      log.debug("stop"); 
      t.stop();
      ```

    - ```
      11:49:42.915 c.TwoPhaseTermination [监控线程] - 将结果保存 
      11:49:43.919 c.TwoPhaseTermination [监控线程] - 将结果保存 
      11:49:44.919 c.TwoPhaseTermination [监控线程] - 将结果保存 
      11:49:45.413 c.TestTwoPhaseTermination [main] - stop 
      11:49:45.413 c.TwoPhaseTermination [监控线程] - 料理后事
      ```

  - 注意：interrupted方法判断完成之后，会自动清除打断标记；isInterrupted不会自动清除

## 主线程和守护线程

- 默认情况下，Java进程会等待所有线程都运行结束了，才会结束。

- 有一种特殊线程叫守护线程，只要其他非守护线程运行结束了，即使守护线程的代码没有执行完，也会强制结束

  - ```java
    log.debug("开始运行...");
    Thread t1 = new Thread(() -> {
    	log.debug("开始运行..."); 
      sleep(2); 
      log.debug("运行结束...");
    }, "daemon");
    // 设置该线程为守护线程 t1.setDaemon(true); t1.start();
    sleep(1); 
    log.debug("运行结束...");
    ```

  - ```
    08:26:38.123 [main] c.TestDaemon - 开始运行... 
    08:26:38.213 [daemon] c.TestDaemon - 开始运行... 
    08:26:39.215 [main] c.TestDaemon - 运行结束...
    // t1不打印运行结束...
    ```

  - 垃圾回收器线程就是一种守护线程

  - Tomcat 中的 Acceptor 和 Poller 线程都是守护线程，所以 Tomcat 接收到 shutdown 命令后，不会等待它们处理完当前请求

## 线程的五种状态（OS层面）

![image-20210226170620349](/Users/i531515/Library/Application Support/typora-user-images/image-20210226170620349.png)

- 【初始状态】仅仅在语言层面创建了线程对象，还没有和操作系统关联
- 【可运行状态】\(就绪状态\)指该线程已经被创建(与操作系统线程关联)，可以由 CPU 调度执行

- 【运行状态】指获取了 CPU 时间片运行中的状态
  - 当 CPU 时间片用完，会从【运行状态】转换至【可运行状态】，会导致线程的上下文切换
- 【阻塞状态】
  - 如果调用了阻塞 API，如 BIO 读写文件，这时该线程实际不会用到 CPU，会导致线程上下文切换，进入 【阻塞状态】
  - 等 BIO 操作完毕，会由操作系统唤醒阻塞的线程，转换至【可运行状态】
  - 与【可运行状态】的区别是，对【阻塞状态】的线程来说只要它们一直不唤醒，调度器就一直不会考虑调度它们

- 【终止状态】表示线程已经执行完毕，生命周期已经结束，不会再转换为其它状态

## 线程的六种状态（Java层面）

![image-20210226171036748](/Users/i531515/Library/Application Support/typora-user-images/image-20210226171036748.png)

- NEW 线程刚被创建，但是还没有调用 start() 方法

- RUNNABLE 当调用了 start() 方法之后，**注意，Java API 层面的 RUNNABLE 状态涵盖了操作系统层面的**

  **【可运行状态】、【运行状态】和【阻塞状态】(由于 BIO 导致的线程阻塞，在 Java 里无法区分，仍然认为是可运行)**

- BLOCKED ， WAITING(join) ， TIMED_WAITING(sleep) 都是 Java API 层面对【阻塞状态】的细分，后面会在状态转换一节详述

- TERMINATED 当线程代码运行结束

- 状态转换过程

  1. new->start: 调用start方法
  2. runnable->waiting: 
     - synchronzied对象如果拥有锁时，调用wait方法，自动从runnable到waiting
     - **当前线程**调用t.join()方法，**当前线程**waiting
     - 调用lock support 方法
  3. waiting->runnable
     - entrySet 中的线程，被notify了，并且和阻塞队列中的其他资源竞争成功了
     - t线程运行结束，或者调用了**当前线程**的interrupt()方法
     - 调用unpark方法
  4. waiting->timed runnable
     - 比到runnable多了sleep方法

# 共享模型之管程





## wait&notify

### sleep和wait的不同

- sleep是thread的方法，wait是object的方法
- sleep不需要配合synchronized使用，wait需要配合synchronized使用
- sleep在睡眠的时候不需要释放对象锁，wait需要

### sleep和wait的相同

- 进入的状态相同，都是timed_wait状态

### 设计模式——同步模式之保护性暂停

- Guarded Suspension
- 有一个结果需要从一个线程传递到另一个线程，让他们关联同一个 GuardedObject
- 如果有结果不断从一个线程到另一个线程那么可以使用消息队列(见生产者/消费者)
- JDK 中，join 的实现、Future 的实现，采用的就是此模式
- 因为要等待另一方的结果，因此归类到同步模式
- ![](https://i.bmp.ovh/imgs/2021/03/260131d93b1f6236.png)

- ```java
  class GuardedObject { 
    private Object response;
  	private final Object lock = new Object();
  	public Object get() { 
      synchronized (lock) { // 条件不满足则等待
  			while (response == null) { 
          try {
  					lock.wait();
  				} catch (InterruptedException e) {
  					e.printStackTrace(); 
          }
  			}
  			return response;
  		}
   	}
  	public void complete(Object response) { 
      synchronized (lock) {
  			// 条件满足，通知等待线程 
        this.response = response; 
        lock.notifyAll();
  		} 
    }
  }
  ```

## ReEntrantLock