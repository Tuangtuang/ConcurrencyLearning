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