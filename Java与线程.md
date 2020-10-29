## Java与线程

### 一、线程的实现

​	线程是比进程更轻量级的调度执行单位。

​	线程可以把一个进程的资源分配和执行调度分开，各个线程既可以共享进程资源（内存地址、文件I/O等），又可以独立调度（线程是CPU调度的基本单位）。

​	实现线程主要有3种方式：使用内核线程实现、使用用户线程实现和使用用户线程加轻量级进程混合实现。

#### 	1、使用内核线程实现

​	内核线程（Kernel-Level Thread,KLT）就是直接由操作系统内核（Kernel，下称内核）支持的线程

​	由内核来完成线程切换

​	通过操纵调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上

​	程序一般不会直接去使用内核线程

​	<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201028153335801.png" alt="image-20201028153335801" style="zoom:67%;" />

#### 	2、使用用户线程实现

​	用户线程指的是完全建立在用户空间的线程库上，系统内核不能感知线程存在的实现

​	用户线程的建立、同步、销毁和调度完全在用户态中完成，不需要内核的帮助

![image-20201028153552099](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201028153552099.png)

#### 	3、使用用户线程加轻量级进程混合实现

​	内核线程与用户线程一起使用的实现

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201028153735263.png" alt="image-20201028153735263" style="zoom:67%;" />

#### 	4、Java线程实现方式

​	基于称为“绿色线程”（Green Threads）的用户线程实现的

​	线程模型替换为基于操作系统原生线程模型来实现

​	操作系统支持怎样的线程模型，在很大程度上决定了Java虚拟机的线程是怎样映射的

### 二、Java线程调度

​	系统为线程分配处理器使用权的过程

​	主要分两种方式：协同式线程调度（Cooperative  Threads-Scheduling）和抢占式线程调度（Preemptive  Threads-Scheduling）。

#### 	1、协同式线程调度（Cooperative  Threads-Scheduling）

​	线程的执行时间由线程本身来控制，线程把自己的工作执行完了之后，要主动通知系统切换到另外一个线程上

#### 	2、抢占式线程调度（Preemptive  Threads-Scheduling）

​	每个线程将由系统来分配执行时间，线程的切换不由线程本身来决定（在Java中，Thread.yield（）可以让出执行时间，但是要获取执行时间的话，线程本身是没有什么办法的）。

​	Java语言一共设置了10个级别的线程优先级（Thread.MIN_PRIORITY至Thread.MAX_PRIORITY），在两个线程同时处于Ready状态时，优先级越高的线程越容易被系统选择执行。

​	线程优先级并不是太靠谱，原因是Java的线程是通过映射到系统的原生线程上来实现的。

### 三、状态转换

#### 	Java语言定义了5种线程状态，在任意一个时间点，一个线程只能有且只有其中的一种状态。

#### 	1、新建（New）：创建后尚未启动的线程处于这种状态。

#### 	2、运行（Runable）：Runable包括了操作系统线程状态中的Running和Ready，也就是处于此状态的线程有可能正在执行，也有可能正在等待着CPU为它分配执行时间。

#### 	3、无限期等待（Waiting）：处于这种状态的线程不会被分配CPU执行时间，它们要等待被其他线程显式地唤醒。
​	**以下方法会让线程陷入无限期的等待状态：**

​		**没有设置Timeout参数的Object.wait（）方法。**

​		**没有设置Timeout参数的Thread.join（）方法。**

​		**LockSupport.park（）方法。**

#### 	4、限期等待（Timed Waiting）：处于这种状态的线程也不会被分配CPU执行时间，不过无须等待被其他线程显式地唤醒，在一定时间之后它们会由系统自动唤醒。

​	**以下方法会让线程进入限期等待状态：**

​		**Thread.sleep（）方法。**

​		**设置了Timeout参数的Object.wait（）方法。**

​		**设置了Timeout参数的Thread.join（）方法。**

​		**LockSupport.parkNanos（）方法。**

​		**LockSupport.parkUntil（）方法。**

#### 	5、阻塞（Blocked）：线程被阻塞了。

​		“阻塞状态”与“等待状态”的区别是：

​		“阻塞状态”在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生；

​		“等待状态”则是在等待一段时间，或者唤醒动作的发生。在程序等待进入同步区域的时候，线程将进入这种状态。

#### 	6、结束（Terminated）：已终止线程的线程状态，线程已经结束执行。

​	<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201028155509478.png" alt="image-20201028155509478" style="zoom:67%;" />

## 线程安全与锁优化

### 一、线程安全

​	当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象是线程安全的。---《Java Concurrency In Practice》/Brian Goetz

#### 	Java语言中的线程安全

##### 	1、不可变

​	Java语言中，只要是有final关键字修饰这个基本数据类型，那么这个基本数据类型就是不可变的。

```java
java.lang.String;//不可对象
//其中的subString();replace();councat();这些方法不会改变原来的字符串的值，而是回了新构造的字符串
```

​	为了保证不影响自己的状态，其中最简单的方法就是将带有状态的变量声明为final，例如java.lang.Integer

```java
//其中类似还有String，Number部分子类，以及BigInteger等大数据类型
private fianl int value;
public Integer(int value){
    this.value=value;
}
```

​	但是Number的子类型的原子类AtomicInteger和AtomicLong则并非不可变的：

```java
java.util.concurrent.atomic;
private volatile int value;
public AtomicInteger(int value){
    this.value = value;
}
public AtomicInteger(){
    
}
```

​	我们主要到这个类的变量状态类型是由volatile修饰的，而volatile本省就是线程不安全的，其修饰的变量值会被修改为意向不到的值。

​	那么被volatile修饰的变量值具有两个属性：

​	1、保证此变量对多有的线程可见。当一个线程修改了变量值，volatile保证新的值能立即同步到主内存，而普通的变量值在线程间传递据需要内存通过主内存。

​	2、禁止指令重拍优化。

##### 	2、绝对线程安全

​	不管运行时环境如何，调用者都不需要任何额外的同步措施。

​	例如java.util.Vector就是一个线程安全的容器：

```java
java.util.Vector;
public synchronized int indexOf(Object o, int index){
    //do something
}
public synchronized int lastIndexOf(Object o){
    //do something
} 
... ...
```

​	我们可以看到在Vector中所有的方法都是用synchronized修饰的，尽管这样效率很低，但是这样确保是线程安全的。但是，即使它所有的方法都被修饰成同步，也不意味着调用它的时候永远都不再需要同步手段了。

Vector测试：

```java
private static Vector<Integer> vector=new Vector<Integer>();
public static void main(String[]args){
    while(true){
        for(int i=0;i<10;i++){
            vector.add(i);
        }
        Thread removeThread=new Thread(() -> {
            for(int i=0;i<vector.size();i++){
                vector.remove(i);
            }
        });
        Thread printThread=new Thread(() -> {
            for(int i=0;i<vector.size();i++){
                System.out.println((vector.get(i)));
            }
        });
        removeThread.start();
        printThread.start();
        //不要同时产生过多的线程，否则会导致操作系统假死
     	while(Thread.activeCount()>20);
    }
}
```

​	运行结果：

```java
Exception in thread "Thread-187299" java.lang.ArrayIndexOutOfBoundsException: Array index out of range: 12
	at java.util.Vector.get(Vector.java:748)
	at com.cxg.weChat.core.config.test.lambda$main$2(test.java:29)
	at com.cxg.weChat.core.config.test$$Lambda$2/10654886.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
```

​	我们可以看到在多线程的环境中，如果不在方法调用端做额外的同步措施的话，使用这段代码仍然是不安全的，因为如果另一个线程恰好在错误的时间里删除了一个元素，导致序号i已经不再可用的话，再用i访问数组就会抛出一个ArrayIndexOutOfBoundsException。

​	为了确保Vector在多线程环境下正常执行，我们将上面的代码做如下修改：

```java
private static Vector<Integer> vector=new Vector<Integer>();
public static void main(String[]args){
    while(true){
        for(int i=0;i<10;i++){
            vector.add(i);
        }
        Thread removeThread=new Thread(() -> {
            synchronized (vector){
                for(int i=0;i<vector.size();i++){
                    vector.remove(i);
                }
            }
        });
        Thread printThread=new Thread(() -> {
            synchronized (vector){
                for(int i=0;i<vector.size();i++){
                    System.out.println((vector.get(i)));
                }
            }
        });
        removeThread.start();
        printThread.start();
        //不要同时产生过多的线程，否则会导致操作系统假死
        while(Thread.activeCount()>20);
    }
}
```

​	这样只修改后执行就不会出现以上的错误，保证线程安全执行。

##### 	3、相对线程安全

​	相对线程安全实际上就是我们所说的线程安全，它需要保证对这个对象单独的操作是线程安全的，我们在调用的时候不需要做额外的保障措施。

​	在Java语言中，大部分线程安全类都是属于这种相对安全的，比如Vector、HashTable、Collection的synchronizedCollection()方法包装的集合等。

##### 	4、线程兼容

​	指对象本身并不是线程安全的，但是可以通过在调用端正确地使用同步手段来保证对象在并发环境中可以安全地使用。平常说一个类不是线程安全的，绝大多数时候指的是这一种情况。

​	Java  API中大部分的类都是属于线程兼容的，如与前面的Vector和HashTable相对应的集合类ArrayList和HashMap等。

##### 	5、线程对立

​	指无论调用端是否采取了同步措施，都无法在多线程环境中并发使用的代码。

​	Java语言天生就具备多线程特性，线程对立这种排斥多线程的代码是很少出现的，而且通常都是有害的，应当尽量避免。

### 二、线程安全的实现方法

##### 	1、同步互斥

​	常见的一种并发正确性保障手段

​	同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一个（或者是一些，使用信号量的时候）线程使用；

​	互斥是实现同步的一种手段，临界区（CriticalSection）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现方式；

​	**互斥是因，同步是果；互斥是方法，同步是目的。**

​	Java中最基本的互斥同步手段就是**synchronized**关键字；

​	synchronized关键字经过编译之后，会在同步块的前后分别形成**monitorenter和monitorexit**这两个字节码指令，这两个字节码都需要一个reference类型的参数来指明要锁定和解锁的对象。

​	如果Java程序中的synchronized明确指定了对象参数，那就是这个对象的reference；

​	如果没有明确指定，那就根据synchronized修饰的是实例方法还是类方法，去取对应的对象实例或Class对象来作为锁对象。

​	在执行monitorenter指令的时候，首先要尝试获取对象的锁，如果这个对象没有被锁定，或者当前的线程已经拥有这个对象的锁，那么锁的计数器就会加1；

​	在执行monitorexit指令的时候，将锁的计数器减1，当计数器为0时，锁释放；

​	如果获取对象锁失败，当前线程就会阻塞，直到这个对象锁被另一个线程释放为止；

​	**虚拟机规范对monitorenter和monitorexit的行为描述需要注意两点：**

​		**1、synchronized同步块对同一条线程来说是可重入的，不会出现自己把自己锁死的问题；**

​		**2、同步块在已进入的线程执行完之前，会阻塞后面其他线程的进入；**

​	Java的线程是映射在系统内存上的，所以在线程转换的时候比较消耗处理器是时间的，特别是被synchronized修饰的getter、setter方法，状态转换消耗的时间有可能比用户执行代码的时间还要长，因此可以得出结论：synchronized是Java语言中重量级锁操作。所以使用要慎重。但是Java在JDK1.6以后对synchronize的进行了优化，因为虚拟机在未来的性能改进中肯定也会更加偏向于原生的synchronized，所以还是提倡在synchronized能实现需求的情况下，优先考虑使用synchronized来进行同步。

​	在JDK1.5之前我们可以使用java.util.concurrent包中的ReentrantLock来实现同步，其用法上和synchronized用法基本类似；

​	ReentrantLock是API层面的互斥锁（lock()/unlock()方法配合try/finally配合使用）；

​	synchronized是原生语言支持的互斥锁；

​	ReentrantLock高级功能：等待可中断、可实现公平锁，以及锁可以绑定多个条件；

##### 	2、非阻塞同步

​	互斥同步最主要的问题就是线程的阻塞和唤醒所带来的的性能问题；

​	因此互斥同步也被称为阻塞同步；

​	从处理问题的方式上说，互斥同步属于一种悲观策略，总是认为只要不去做正确的同步措施就会出错，无论共享数据是否真的存在竞争，都要加锁，这就是为什么有些程序将synchronized称为悲观锁的原因；

​	那么我们解决这问题应发的解决方式就是乐观策略，也被称为乐观锁：先进行操作，如果没有其他线程争用共享数据，那操作就成功了；如果共享数据有争用，产生了冲突，那就再采取其他的补偿措施（最常见的补偿措施就是不断地重试，直到成功为止）；

##### 	3、无同步方案

​	要保证线程安全，并不是一定就要进行同步，两者没有因果关系；

​	比如可重入代码；线程本地存储；

## 锁优化

### 	一、自旋锁与自适应自旋

​	自旋锁：让线程等待，我们只需让线程执行一个忙循环（自旋）；

​	自旋锁的优缺点：如果锁被占用的时间很短，自旋等待的效果就会非常好，反之，如果锁被占用的时间很长，那么自旋的线程只会白白消耗处理器资源，而不会做任何有用的工作，反而会带来性能上的浪费。

​	自适应自旋锁：自适应意味着自旋的时间不再固定了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定；

### 	二、锁消除

​	指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除；

​	主要判定依据来源于逃逸分析的数据支持；

### 	三、锁粗化

### 	四、轻量级锁

​	轻量级锁并不是用来代替重量级锁的；

​	相对于重量级锁而言；

​	HotSpot虚拟机的对象头（Object Header）分为两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄（Generational GC Age）等，这部分数据的长度在32位和64位的虚拟机中分别为32bit和64bit，官方称它为“Mark Word”，它是实现轻量级锁和偏向锁的关键。![image-20201029161116719](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201029161116719.png)

​	轻量级锁的执行过程：

​	1、轻量级锁CAS操作之前堆栈与对象的状态

![image-20201029161957317](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201029161957317.png)

​	2、轻量级锁CAS操作之后堆栈与对象的状态

![image-20201029161506835](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201029161506835.png)

### 	五、偏向锁

​	在无竞争的情况下把整个同步都消除掉，连CAS操作都不做了。

​	偏向锁、轻量级锁的状态转化及对象Mark Word的关系：

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\image-20201029162139861.png" alt="image-20201029162139861" style="zoom:67%;" />