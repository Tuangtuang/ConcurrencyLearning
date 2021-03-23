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

## 临界区 Critical Section

- 一段代码块，如果存在对共享资源的多线程读写操作，称这段代码块为临界区。

## 竞态条件 Race Condition

- 多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件

## 常见线程安全类

- String
- Integer
- StringBuffer
- Random
- Vector
- Hashtable java.util.concurrent 包下的类

## synchronized 解决方案

- 为了避免临界区的竞态条件发生，有多种手段可以达到目的。
  - 阻塞式的解决方案:synchronized，Lock 
  - 非阻塞式的解决方案:原子变量

### synchronized语法

```java
synchronized(对象) // 线程1， 线程2(blocked) {
		临界区
}
```

- sample

  ```java
  static int counter = 0;
  static final Object room = new Object();
  public static void main(String[] args) throws InterruptedException { 
    Thread t1 = new Thread(() -> {
  		for (int i = 0; i < 5000; i++) { 
        synchronized (room) {
  				counter++; 
        }
  		}
  	}, "t1");
  	Thread t2 = new Thread(() -> {
  		for (int i = 0; i < 5000; i++) {
  			synchronized (room) { 
          counter--;
  			} 
    	}
  	}, "t2");
  	t1.start();
  	t2.start();
  	t1.join();
  	t2.join(); log.debug("{}",counter);
  }
  ```
  - 如果把 synchronized(obj) 放在 for 循环的外面，如何理解?
  - 如果 t1 synchronized(obj1) 而 t2 synchronized(obj2) 会怎样运作?
  - 如果 t1 synchronized(obj) 而 t2 没有加会怎么样?如何理解?

- 面向对象优化

  - ```java
    class Room {
    	int value = 0;
    	public void increment() { 
        synchronized (this) {
    				value++; 
        }
    	}
    	public void decrement() { 
        synchronized (this) {
    			value--; 
        }
    	}
      public int get() { 
        synchronized (this) {
    			return value; 
        }
    	} 
    }
    
    @Slf4j
    public class Test1 {
    	public static void main(String[] args) throws InterruptedException { 
        Room room = new Room();
    		Thread t1 = new Thread(() -> {
    			for (int j = 0; j < 5000; j++) { 
            room.increment();
          }
    		}, "t1");
    		Thread t2 = new Thread(() -> {
    			for (int j = 0; j < 5000; j++) {
    				room.decrement(); 
       	 }
    		}, "t2"); 
      	t1.start(); 
        t2.start();
    		t1.join();
    		t2.join();
    		log.debug("count: {}" , room.get());
    	} 
    }
    ```

- 方法上的synchronized

  - ```java
    class Test{
    	public synchronized void test() {
    	} 
    }
    // 等价于
    class Test{
    	public void test() {
    	synchronized(this) { }
    	} 
    }
    ```

  - ```java
    class Test{
    	public synchronized static void test() {
    	} 
    }
    //等价于
    class Test{
    	public static void test() {
    		synchronized(Test.class) { }
    	} 
    }
    ```

### synchronized原理

#### Java对象头

- 普通对象

  ```
   |--------------------------------------------------------------| 
   | 												Object Header (64 bits) 							| 
   |------------------------------------|-------------------------| 
   | Mark Word (32 bits)							 	| 	Klass Word (32 bits)  | 
   |------------------------------------|-------------------------|
  ```
  - Klass指针指向了类对象

- 数组对象

  ```
   |---------------------------------------------------------------------------------| 
   | 																Object Header (96 bits) 												 | 
   |--------------------------------|-----------------------|------------------------| 
   | Mark Word(32bits)						 	| Klass Word(32bits) 		| array length(32bits) 	 | 
   |--------------------------------|-----------------------|------------------------|
  ```

- Mark Word 结构为

  ```
  |-------------------------------------------------------|--------------------| 
  | Mark Word (32 bits) 																	| 						 State | 
  |-------------------------------------------------------|--------------------| 
  | hashcode:25 | age:4 | biased_lock:0 | 01 							| Normal 						 | 
  |-------------------------------------------------------|--------------------| 
  | thread:23 | epoch:2 | age:4 | biased_lock:1 | 01 			| Biased 						 | 
  |-------------------------------------------------------|--------------------| 
  | ptr_to_lock_record:30 | 00 														| Lightweight Locked | 
  |-------------------------------------------------------|--------------------| 
  | ptr_to_heavyweight_monitor:30 | 10 										| Heavyweight Locked | 
  |-------------------------------------------------------|--------------------| 
  | 																									|11 | MarkedforGC 	 		 |		 
  |-------------------------------------------------------|--------------------|
  ```

  - 01表示normal或者开了偏向锁（再往前看一位，区别是否开启偏向锁）
  - 10表示中重量锁
  - 00表示轻量锁

#### Monitor锁/管程

- 每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁(重量级)之后，该对象头的

  Mark Word 中就被设置指向 Monitor 对象的指针

  ![image-20210304150022244](/Users/i531515/Library/Application Support/typora-user-images/image-20210304150022244.png)

- synchronized 的对象中的对象头，最后两位状态下01变10，并且ptr_to_heavyweight_monitor设置成指向monitor的指针
- 刚开始 Monitor 中 Owner 为 null
- 当 Thread-2 执行 synchronized(obj) ，先检查obj有没有关联monitor，就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一 个 Owner
- 在 Thread-2 上锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行 synchronized(obj)，就会进入 EntryList BLOCKED
- Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争的时是非公平的
- 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面讲 wait-notify 时会分析
- synchronized 必须是进入同一个对象的 monitor 才有上述的效果
- 不加 synchronized 的对象不会关联监视器，不遵从以上规则

### synchronized 原理进阶

#### 轻量级锁

- 为什么要使用轻量级锁？
  
- 因为重量级锁要用到OS提供的monitor，开销大
  
- 使用场景

  - 如果一个对象虽然有多线程要加锁，但是加锁时间是错开的（没有竞争），那么就可以用加轻量级锁。
  - 轻量级锁对使用者是透明的，即语法仍然是 synchronized

- sample

  - ```java
    static final Object obj = new Object(); 
    public static void method1() {
    	synchronized( obj ) { 
        // 同步块 A
    		method2(); 
      }
    }
    public static void method2() {
      synchronized( obj ) { 
        // 同步块 B
    	} 
    }
    ```

- 创建锁记录(Lock Record)对象，每个线程都的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的 Mark Word

  - ![image-20210304152745695](/Users/i531515/Library/Application Support/typora-user-images/image-20210304152745695.png)

- 让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换 Object 的 Mark Word，将 Mark Word 的值存 入锁记录
  
- ![image-20210304152857478](/Users/i531515/Library/Application Support/typora-user-images/image-20210304152857478.png)
  
- 如果 cas 替换成功，对象头中存储了 锁记录地址和状态 00（表示轻量级锁） ，表示由该线程给对象加锁，这时图示如下
  
- ![image-20210304152954795](/Users/i531515/Library/Application Support/typora-user-images/image-20210304152954795.png)
  
- 如果 cas 失败，有两种情况
  - 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程
  - 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数
    - ![image-20210304153030297](/Users/i531515/Library/Application Support/typora-user-images/image-20210304153030297.png)
- 当退出 synchronized 代码块(解锁时)如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重 入计数减一
  
- ![image-20210304153108639](/Users/i531515/Library/Application Support/typora-user-images/image-20210304153108639.png)
  
- 当退出 synchronized 代码块(解锁时)锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象 头
  - 成功，解锁成功
  - 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程

#### 锁膨胀

- 将轻量级锁升级成重量级锁
- 如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁(有竞争)，这时需要进行锁膨胀，将轻量级锁变为重量级锁。

- 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁

  ![image-20210304154413229](/Users/i531515/Library/Application Support/typora-user-images/image-20210304154413229.png)

- 这时 Thread-1 加轻量级锁失败，进入锁膨胀流程
  - 即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址
  - 然后自己进入 Monitor 的 EntryList BLOCKED，将owner设置成thread0
  - ![image-20210304154629181](/Users/i531515/Library/Application Support/typora-user-images/image-20210304154629181.png)

- 当 Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁 流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程

#### 自旋优化

- **多核CPU下才能进行**

- **重量级**锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功(即这时候持锁线程已经退出了同步块，释放了锁)，这时当前线程就可以避免阻塞。（为什么要避免阻塞？阻塞需要上下文切换，代价比较大）

  - 成功情况
    - ![image-20210304155225756](/Users/i531515/Library/Application Support/typora-user-images/image-20210304155225756.png)

  - 失败情况
    - ![image-20210304155333932](/Users/i531515/Library/Application Support/typora-user-images/image-20210304155333932.png)

  - 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会 高，就多自旋几次;反之，就少自旋甚至不自旋，总之，比较智能。
    - Java 7 之后不能控制是否开启自旋功能

#### 偏向锁

- 轻量级锁的缺点：执行锁重入时，仍然需要CAS操作

- Java 6 中引入了偏向锁来做进一步优化:只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现 这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有

  - ![image-20210304160725423](/Users/i531515/Library/Application Support/typora-user-images/image-20210304160725423.png)

  - ![image-20210304160744636](/Users/i531515/Library/Application Support/typora-user-images/image-20210304160744636.png)

- ```
  |-------------------------------------------------------|--------------------| 
  | Mark Word (32 bits) 																	| 						 State | 
  |-------------------------------------------------------|--------------------| 
  | hashcode:25 | age:4 | biased_lock:0 | 01 							| Normal 						 | 
  |-------------------------------------------------------|--------------------| 
  | thread:23 | epoch:2 | age:4 | biased_lock:1 | 01 			| Biased 						 | 
  |-------------------------------------------------------|--------------------| 
  | ptr_to_lock_record:30 | 00 														| Lightweight Locked | 
  |-------------------------------------------------------|--------------------| 
  | ptr_to_heavyweight_monitor:30 | 10 										| Heavyweight Locked | 
  |-------------------------------------------------------|--------------------| 
  | 																									|11 | MarkedforGC 	 		 |		 
  |-------------------------------------------------------|--------------------|
  ```

  - 如果开启了偏向锁(默认开启)，那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的 thread、epoch、age 都为 0
  - 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加VM参数 - XX:BiasedLockingStartupDelay=0 来禁用延迟
  - 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、 age 都为 0，第一次用到 hashcode 时才会赋值

- 撤销

  - 调用对象 hashCode
    - 调用了对象的 hashCode，但偏向锁的对象 MarkWord 中存储的是线程 id，如果调用 hashCode 会导致偏向锁被撤销（因为存不下了）
      - 轻量级锁会在锁记录中记录 hashCode
      - 重量级锁会在 Monitor 中记录 hashCode

  - 其它线程使用对象
    - 当有其它线程使用偏向锁对象时，会将偏向锁升级为轻量级锁
  - wait/notify

- 批量重定向

  - **如果对象虽然被多个线程访问，但没有竞争，**这时偏向了线程 T1 的对象仍有机会重新偏向 T2，重偏向会重置对象的 Thread ID

  - 当撤销偏向锁阈值超过 20 次后，jvm 会这样觉得，我是不是偏向错了呢，于是会在给这些对象加锁时重新偏向至加锁线程

  - 当撤销偏向锁阈值超过 40 次后，jvm 会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象 都会变为不可偏向的，新建的对象也是不可偏向的

#### 锁消除

- sample

  ```java
  @Fork(1) 
  @BenchmarkMode(Mode.AverageTime) 
  @Warmup(iterations=3) 
  @Measurement(iterations=5) 
  @OutputTimeUnit(TimeUnit.NANOSECONDS) 
  public class MyBenchmark {
  	static int x = 0;
  	@Benchmark
  	public void a() throws Exception {
  		x++; 
    }
  	@Benchmark
  	public void b() throws Exception { 
      Object o = new Object(); 
      synchronized (o) {
  			x++;
  		} 
    }
  }
  ```

- JIT即使编译，发现了o是局部变量，不会发生资源竞争和共享，所以位自动优化消除锁



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

## ReEntrantLock VS Synchronize

- Reentrantlock相比synchronize有如下特点

  - 可中断

    - 在等待锁的过程中可以被打断，会抛出异常

  - 可设置超时时间

  - 可设置为公平锁

  - 支持多个条件变量

    - synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待 ReentrantLock 的条件变量比 synchronized 强大之处在于，它是**支持多个条件变量**的，这就好比

      - synchronized 是那些不满足条件的线程都在一间休息室等消息，而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤 醒

      - 使用要点:
        - await 前需要获得锁
        - await 执行后，会释放锁，进入 conditionObject 等待 await 的线程被唤醒(或打断、或超时)取重新竞争 lock 锁 竞争 lock 锁成功后，从 await 后继续执行

- 和synchronized一样都支持重入

# 共享模型之volatile

## Java内存模型

- JMM 即 Java Memory Model，它定义了主存、工作内存抽象概念，底层对应着 CPU 寄存器、缓存、硬件内存、 CPU 指令优化等。

- JMM 体现在以下几个方面
  - 原子性 - 保证指令不会受到线程上下文切换的影响 
  - 可见性 - 保证指令不会受 cpu 缓存的影响
  - 有序性 - 保证指令不会受 cpu 指令并行优化的影响

## Volatile

- 补充
- 

# 共享模型之无锁

## CAS

- compare and set
  - ![image-20210310103336920](/Users/i531515/Library/Application Support/typora-user-images/image-20210310103336920.png)
  - 其实 CAS 的底层是 lock cmpxchg 指令(X86 架构)，在单核 CPU 和多核 CPU 下都能够保证【比较-交 换】的原子性。

## CAS和Volatile

- 获取共享变量时，为了保证该变量的可见性，需要使用 volatile 修饰。它可以用来修饰成员变量和静态成员变量，他可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取 它的值，线程操作 volatile 变量都是直接操作主存。即一个线程对 volatile 变量的修改，对另一个线程可见。

- volatile 仅仅保证了共享变量的可见性，让其它线程能够看到最新值，但不能解决指令交错问题(不能保证原子性)

- CAS 必须借助 volatile 才能读取到共享变量的最新值来实现【比较并交换】的效果

## 为什么无锁并发效率高

- 无锁情况下，即使重试失败，线程始终在高速运行，没有停歇，而 synchronized 会让线程在没有获得锁的时候，发生上下文切换，进入阻塞。
- 但无锁情况下，因为线程要保持运行，需要额外 CPU 的支持，CPU 在这里就好比高速跑道，没有额外的跑道，线程想高速运行也无从谈起，虽然不会进入阻塞，但由于没有分到时间片，仍然会进入可运行状态，还会导致上下文切换。

## CAS 的特点

- 结合 CAS 和 volatile 可以实现无锁并发，适用于**线程数少、多核 CPU** 的场景下。
- CAS 是基于**乐观锁**的思想:最乐观的估计，不怕别的线程来修改共享变量，就算改了也没关系，我吃亏点再重试呗。

- synchronized 是基于**悲观锁**的思想:最悲观的估计，得防着其它线程来修改共享变量，我上了锁你们都别想 改，我改完了解开锁，你们才有机会。

- CAS 体现的是无锁并发、无阻塞并发，请仔细体会这两句话的意思

- 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提升的因素之一 但如果竞争激烈，可以想到重试必然频繁发生，反而效率会受影响

# 不可变对象

## Final关键字

### 设置final原理

- ```java
    public class TestFinal { 
      final int a = 20;
    	}
  ```

- ```
  0: aload_0
  1: invokespecial #1 
  4: aload_0
  5: bipush 20 
  7: putfield #2
  <-- 写屏障 
  10: return
  ```

- 写屏障保证了之前的指令不会在写屏障之后执行

- 保证了其他线程对变量的可见性，保证在其它线程读到 它的值时不会出现为 0 的情况（初始化之前变量是0）

# ThreadPoolExecutor

![image-20210310104733209](/Users/i531515/Library/Application Support/typora-user-images/image-20210310104733209.png)

## 线程池状态

- ThreadPoolExecutor 使用 int 的高 3 位来表示线程池状态，低 29 位表示线程数量
- ![image-20210310113652758](/Users/i531515/Library/Application Support/typora-user-images/image-20210310113652758.png)

- 从数字上比较，TERMINATED > TIDYING > STOP > SHUTDOWN > RUNNING

- 这些信息存储在一个原子变量 ctl 中，目的是将线程池状态与线程个数合二为一，**这样就可以用一次 cas 原子操作**

  **进行赋值**

## 构造方法

```java
public ThreadPoolExecutor(int corePoolSize, 
                          int maximumPoolSize,
													long keepAliveTime,
													TimeUnit unit, 
                          BlockingQueue<Runnable> workQueue, 
                          ThreadFactory threadFactory, 
                          RejectedExecutionHandler handler)
```

- corePoolSize 核心线程数目 (最多保留的线程数) 
- maximumPoolSize 最大线程数目 
- keepAliveTime 生存时间 - 针对救急线程
- unit 时间单位 - 针对救急线程
- workQueue 阻塞队列
- threadFactory 线程工厂 - 可以为线程创建时起个好名字 
- handler 拒绝策略

## 工作方式

- 线程池中刚开始没有线程，当一个任务提交给线程池之后，线程池会创建一个新线程来执行任务。
- 当线程数达到corePoolSize 并没有线程空闲，这时再加入任务，新加的任务会被加入workQueue 队列排队，直到有空闲的线程。
- 如果队列选择了有界队列，那么任务超过了队列大小时，会创建 maximumPoolSize - corePoolSize 数目的线程来救急。
- 如果线程到达 maximumPoolSize 仍然有新任务这时会执行拒绝策略。拒绝策略 jdk 提供了 4 种实现：
  - AbortPolicy 让调用者抛出 RejectedExecutionException 异常，这是默认策略
  - CallerRunsPolicy 让调用者运行任务
  - DiscardPolicy 放弃本次任务
  - DiscardOldestPolicy 放弃队列中最早的任务，本任务取而代之

- 当高峰过去后，超过corePoolSize 的救急线程如果一段时间没有任务做，需要结束节省资源，这个时间由 keepAliveTime 和 unit 来控制。
- 根据这个构造方法，JDK Executors 类中提供了众多工厂方法来创建各种用途的线程池
  - ![image-20210310131845470](/Users/i531515/Library/Application Support/typora-user-images/image-20210310131845470.png)

## 四种线程池

- newFixedThreadPool

  - ```java
    public static ExecutorService newFixedThreadPool(int nThreads) { 
    	return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
    }
    ```

  - **核心线程数 == 最大线程数(没有救急线程被创建)**，因此也无需超时时间

  - 阻塞队列是无界的，可以放任意数量的任务

  - 适用于任务量已知，相对耗时的任务

- newCachedThreadPool

  - ```java
    public static ExecutorService newCachedThreadPool() { 
    	return new ThreadPoolExecutor(0, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>());
    }
    ```

  - **核心线程数是 0， 最大线程数是 Integer.MAX_VALUE**，救急线程的空闲生存时间是 60s，意味着

    - 全部都是救急线程(60s 后可以回收)

    - 救急线程可以无限创建

    - 队列采用了 **SynchronousQueue** 实现特点是，它**没有容量**，没有线程来取，任务是放不进去的(一手交钱、一手交货)

    - ```java
      SynchronousQueue<Integer> integers = new SynchronousQueue<>(); 
      new Thread(() -> {
      	try {
      		log.debug("putting {} ", 1); 
          integers.put(1); 
          log.debug("{} putted...", 1);
      		log.debug("putting...{} ", 2); 
          integers.put(2);
      		log.debug("{} putted...", 2);
      	} catch (InterruptedException e) { 
          e.printStackTrace();
      	} 
      },"t1").start();
      sleep(1);
      new Thread(() -> { 
        try {
      		log.debug("taking {}", 1);
      		integers.take();
      	} catch (InterruptedException e) {
      		e.printStackTrace(); 
        }
      },"t2").start();
      sleep(1);
      new Thread(() -> { 
        try {
      		log.debug("taking {}", 2);
      		integers.take();
      	} catch (InterruptedException e) {
      		e.printStackTrace(); }
      },"t3").start();
      ```

    - ```
      11:48:15.500 c.TestSynchronousQueue [t1] - putting 1 
      11:48:16.500 c.TestSynchronousQueue [t2] - taking 1 
      11:48:16.500 c.TestSynchronousQueue [t1] - 1 putted... 
      11:48:16.500 c.TestSynchronousQueue [t1] - putting...2 
      11:48:17.502 c.TestSynchronousQueue [t3] - taking 2 
      11:48:17.503 c.TestSynchronousQueue [t1] - 2 putted...
      ```

    - 整个线程池表现为线程数会根据任务量不断增长，没有上限，当任务执行完毕，空闲 1分钟后释放线程。 **适合任务数比较密集，但每个任务执行时间较短的情况**

- newSingleThreadExecutor

  - ```java
    public static ExecutorService newSingleThreadExecutor() { 
     	return new FinalizableDelegatedExecutorService(new ThreadPoolExecutor(1, 1, 0L, 
                                                                            TimeUnit.MILLISECONDS, 
                                                                            new LinkedBlockingQueue<Runnable>()));
    }
    ```

  - 希望多个任务排队执行。**线程数固定为 1**，**任务数多于 1 时，会放入无界队列排队**。任务执行完毕，这唯一的线程也不会被释放。

  - 和自己创建单线程执行的区别：

    - 自己创建单线程执行，如果失败没有任何补救措施，而线程池会新建一个线程，保证线程池的正常工作

  - 和Executors.newFixedThreadPool(1) 的区别

    - Executors.newSingleThreadExecutor() 线程个数始终为1，不能修改

      - FinalizableDelegatedExecutorService 应用的是装饰器模式，只对外暴露了 ExecutorService 接口，因

      此不能调用 ThreadPoolExecutor 中特有的方法

    - Executors.newFixedThreadPool(1) 初始时为1，以后还可以修改
      - 对外暴露的是 ThreadPoolExecutor 对象，可以强转后调用 setCorePoolSize 等方法进行修改

- newScheduledThreadPool

  - 在『任务调度线程池』功能加入之前，可以使用 java.util.Timer 来实现定时功能，Timer 的优点在于简单易用，但 由于所有任务都是由同一个线程来调度，因此所有任务都是串行执行的，同一时间只能有一个任务在执行，前一个 任务的延迟或异常都将会影响到之后的任务。

  - 执行周期性定时任务

    - ```java
      public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
          return new ScheduledThreadPoolExecutor(corePoolSize);
      }
      public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
      }
      ```

# JUC

## AQS（abstract queued synchronizer）

- 阻塞式同步器框架

- 特点：

  - 用state属性来表示资源的状态（分独占模式和共享模式），子类需要定义如何维护这个状态，控制如何获取锁和释放锁
    - getState - 获取state状态
    - set State - 设置state状态（用CAS）
  - 提供了FIFO的等待队列，而共享模式可以允许多个线程访问
  - 条件变量来实现等待、唤醒机制，支持多个条件变量，类似于 Monitor 的 WaitSet

- 子类主要实现这样一些方法(默认抛出 UnsupportedOperationException)

  - tryAcquire 
  - tryRelease 
  - tryAcquireShared 
  - tryReleaseShared 
  - isHeldExclusively

- 获取锁的姿势

  - ```java
     // 如果获取锁失败
    if (!tryAcquire(arg)) {
    // 入队, 可以选择阻塞当前线程 park unpark 
    }
    ```

  - ```java
     // 如果释放锁成功
    if (tryRelease(arg)) {
    // 让阻塞线程恢复运行 
    }
    ```

    

## Reentrant Lock

### 与synchronize的区别

- 可中断
- 可以设置超时时间
- 可以设置成公平锁
- 支持多个条件变量

### 原理

#### 加锁流程（默认为非公平）

- 没有竞争时
  - 设置state为1，owner为当前线程
  - ![image-20210312111434880](/Users/i531515/Library/Application Support/typora-user-images/image-20210312111434880.png)

- 第一个竞争出现时

  - ![image-20210312111551598](/Users/i531515/Library/Application Support/typora-user-images/image-20210312111551598.png)

  - CAS尝试将state由0改成1，结果失败

  - 进入try Acquire逻辑，这时state仍然是1，结果仍然是1

  - 接下来进入addWaiter逻辑，构造node队列

    - ![image-20210312111750780](/Users/i531515/Library/Application Support/typora-user-images/image-20210312111750780.png)

    - 图中黄色表示该node的wait status状态，其中0位默认正常状态
    - 第一个node为dummy，用来占位置

  - 当前线程进入 acquireQueued 逻辑

    - acquireQueued 会在一个死循环中不断尝试获得锁，失败后进入 park 阻塞
    - 如果自己是紧邻着 head(排第二位)，那么再次 tryAcquire 尝试获取锁，当然这时 state 仍为 1，失败
    - 进入 shouldParkAfterFailedAcquire 逻辑，将前驱 node，即 head 的 waitStatus 改为 -1，这次返回 false
    - shouldParkAfterFailedAcquire 执行完毕回到 acquireQueued ，再次 tryAcquire 尝试获取锁，当然这时 state 仍为 1，失败
    - 当再次进入 shouldParkAfterFailedAcquire 时，这时因为其前驱 node 的 waitStatus 已经是 -1，这次返回 true
    - 进入 parkAndCheckInterrupt， Thread-1 park(灰色表示)

  - Thread-0 释放锁，进入 tryRelease 流程，如果成功

    - 设置 exclusiveOwnerThread 为 null
    - state = 0
      - ![image-20210312112513658](/Users/i531515/Library/Application Support/typora-user-images/image-20210312112513658.png)

    - 当前队列不为 null，并且 head 的 waitStatus = -1，进入 unparkSuccessor 流程，找到队列中离 head 最近的一个 Node(没取消的)，unpark 恢复其运行，本例中即为 Thread-1 回到 Thread-1 的 acquireQueued 流程
      - ![image-20210312112558365](/Users/i531515/Library/Application Support/typora-user-images/image-20210312112558365.png)
      - 如果加锁成功（没有竞争），会设置
        - owner为当前线程，state=1
        - 删除node
      - 如果这时候有其它线程来竞争(非公平的体现)，例如这时有 Thread-4 来了
        - 如果不巧又被 Thread-4 占了先
          - Thread-4 被设置为 exclusiveOwnerThread，state = 1
          - Thread-1 再次进入 acquireQueued 流程，获取锁失败，重新进入 park 阻塞

#### 可重入原理

- 重入时set state++

#### 条件变量实现原理

- 每个条件变量其实就对应着一个等待队列，其实现类是 ConditionObject

## Reentrant Read Write Lock

- 为什么要read write分开
  - 当读操作远高于写操作时，这时候使用读写锁让读-读可以并发，提高性能
- 读锁
  - 读锁**不支持条件变量** **重入时升级不支持**:即持有读锁的情况下去获取写锁，会导致获取写锁永久等待
- 写锁
  - 重入**时降级支持**:即持有写锁的情况下去获取读锁

### 原理

- 读写锁用的是同一个syn同步器，因此等待队列、state也是同个

- t1写锁**（状态为排他锁）**，t2读锁**（状态为shared）**

- t1成功上锁，流程和reentrant lock没有区别，**state上有区别，高16位表示读锁，低16位表示写锁**，
- t2执行读锁，这时进入读锁的sync.acquireShared(1)流程，首先会进入try Acquire Shared流程，如果有写锁占据，那么返回-1，表示失败
  - try Acquire Shared返回值
    - -1表示失败
    - 0表示成功，但是后继节点不会被唤醒
    - 正数表示成功，而且数值是还有几个后续节点要被唤醒，读写锁返回1
  - ![image-20210312130753027](/Users/i531515/Library/Application Support/typora-user-images/image-20210312130753027.png)
  - 因为t1已经加了写锁，所以失败，进入doAcuquireShare流程

## StampedLock

- 该类自 JDK 8 加入，是为了进一步优化读性能，它的特点是在使用读锁、写锁时都必须配合【戳】

## Semaphore 原理

- Semaphore 有点像一个停车场，permits 就好像停车位数量，当线程获得了 permits 就像是获得了停车位，然后 停车场显示空余车位减一

## CountDownLatch & CycilBarrier

### CountDownLatch

- 用来进行线程同步协作，等待所有线程完成倒计时。 其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一
- ountDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。

### CycilBarrier

- 循环栅栏，用来进行线程协作，等待线程满足某个计数。构造时设置『计数个数』，每个线程执 行到某个需要“同步”的时刻调用 await() 方法进行等待，当等待的线程数满足『计数个数』时，继续执行
- 它主要的方法就是一个：await()。await() 方法被调用一次，计数便会减少1，并阻塞住当前线程。当计数减至0时，阻塞解除，所有在此 CyclicBarrier 上面阻塞的线程开始运行。在这之后，如果再次调用 await() 方法，计数就又会变成 N-1，新一轮重新开始，这便是 Cyclic 的含义所在。CyclicBarrier.await() 方法带有返回值，用来表示当前线程是第几个到达这个 Barrier 的线程。

### 区别

- CountdownLatch 计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。
- **CountdownLatch适用于所有线程通过某一点后通知方法,而CyclicBarrier则适合让所有线程在同一点同时执行**
- CountdownLatch利用继承AQS的共享锁来进行线程的通知,利用CAS来进行,而CyclicBarrier则利用ReentrantLock的Condition来阻塞和通知线程
- ![image-20210315114119578](/Users/i531515/Library/Application Support/typora-user-images/image-20210315114119578.png)

## ThreadLocal

ThreadLocal中填充的变量属于**当前**线程，该变量对其他线程而言是隔离的。其使用场景如下：

**1、在进行对象跨层传递的时候，使用ThreadLocal可以避免多次传递，打破层次间的约束。**

**2、线程间数据隔离。**

**3、进行事务操作，用于存储线程事务信息。**

**4、数据库连接，Session会话管理。**

`ThreadLocal`类最重要的几个方法如下：

```java
//获取ThreadLocal的值
public T get() { }
//设置ThreadLocal的值
public void set(T value) { }
//删除ThreadLocal
public void remove() { }
//初始化ThreadLocal的值
protected T initialValue() { }
```

### 原理

- 首先，简单回顾一下，ThreadLocal是一个线程本地变量，每个线程维护自己的变量副本，多个线程互相不可见，因此多线程操作该变量不必加锁，适合不同线程使用不同变量值的场景。其**数据结构**是每个线程Thread类都有个属性ThreadLocalMap，用来维护该线程的多个ThreadLocal变量，该Map是自定义实现的Entry[]数组结构，并非继承自原生Map类，Entry其中Key即是ThreadLocal变量本身，Value则是具体该线程中的变量副本值。
- ThreadLocalMap使用弱引用存储ThreadLocal
  - 假如使用强引用，当ThreadLocal不再使用需要回收时，发现某个线程中ThreadLocalMap存在该ThreadLocal的强引用，无法回收，造成内存泄漏。
  - ![img](https://pic2.zhimg.com/80/v2-e2d4b8eac152596232d3e32313927d59_720w.jpg)

### ThreadLocal 内存泄漏的原因

从上图中可以看出，hreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal不存在外部**强引用**时，Key(ThreadLocal)势必会被GC回收，这样就会导致ThreadLocalMap中key为null， 而value还存在着强引用，只有thead线程退出以后,value的强引用链条才会断掉。

但如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：

> Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value

永远无法回收，造成内存泄漏。

