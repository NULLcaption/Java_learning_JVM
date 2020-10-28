## Java虚拟机内存模型与内存溢出异常

​	如图，Java虚拟机一般在Java程序执行的执行的时候将内存分为若干个不同的数据区域，这些区域有不同的用途以及创建和销毁时间。

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201019092913335.png" alt="image-20201019092913335" style="zoom: 80%;" />

#### 	一、程序计数器

##### 	用于存放下一个指令。

##### 	可以看做是当前线程所执行的字节码的行号指示器。

##### 	字节码解释器工作时就是通过改变这个计数器的值来去下一条需要执行的的字节码指令。

​	Java虚拟机多线程是通过多线程轮流切换并分配处理时间的方式来实现的，在任何一个确定的时刻，一个处理器只能执行一条线程中的指令；因为了保证线程切换后恢复到正确的位置，每一条线程 都需要有一个独立的计数器，来保证各个线程之间的独立运行，独立存储，我们称之为“线程私有内存”。

​	**一般的如果线程执行的是一个Java方法，则这个技术器记录的就是这个正在执行的是虚拟机字节码指令的地址；如果是一个Native方法，则这个计数器的值为空；因此这个内存区域是Java虚拟集中唯一一个没有规定OOM的区域。**

#### 	二、虚拟机栈

##### 	1、线程私有的内存空间，它与Java线程同一时间创建；

##### 	2、保存方法的局部变量、部分结果，参与方法的调用和返回；

​	注意：经常有人把Java内存区分为堆内存（Heap）和栈内存（Stack），这种分法比较粗糙，Java内存区域的划分实际上远比这复杂。这种划分方式的流行只能说明大多数程序员最关注的、与对象内存分配关系最密切的内存区域是这两块。其中所指的“堆”笔者在后面会专门讲述，**而所指的“栈”就是现在讲的虚拟机栈，或者说是虚拟机栈中局部变量表部分。**

​	**局部变量表存放了编译期可知的数据各种基本数据类型（int,boolean,byte,char,float,long,double）,对象应用（reference类型，不同于对象本身，可能是一个指向对象的起始地址的引用指针，也可能是指向一个对象的句柄或者与其他对象相关的位置），和returnAddress类型**

##### 	3、会抛出StackOverflowError和OOM异常；

#### 	三、本地方法栈

##### 	管理本地方法的调用。

​	本地方法栈和虚拟机栈发挥的作用非常相似，只不过**虚拟机栈执行的是Java的字节码服务；而本地方法栈则为虚拟机使用的Native方法服务**。

##### 	会抛出StackOverflowError和OOM异常；

#### 	四、Java堆

##### 		Java运行时最重要的部分，被所有线程共享的一块区域，在虚拟启动时被创建

##### 		Java堆是垃圾收集管理器的主要区域

##### 		几乎所有的对象和数组都在堆中分配空间

##### 		Java堆可以处于物理内存上不连续的内存空间中，只需要逻辑上连续即可；用过-Xmx和——Xms控制

##### 	会抛出OOM异常

##### 		Java堆的分类：

###### 				新生代（eden、From Survivor ，To Survivor）

###### 				中生代

###### 				年老代

#### 	五、方法区

##### 		为JVM中所有线程共享

##### 		主要保存类的元数据信息

##### 		会抛出OOM异常

## HorSpot虚拟的对象管理过程

### 	一、对象创建

```CPP
//确保常量池中存放的是已解释的类
if（！constants-＞tag_at（index）.is_unresolved_klass（））{
	//断言确保是klassOop和instanceKlassOop（这部分下一节介绍）
	oop entry=（klassOop）*constants-＞obj_at_addr（index）；
	assert（entry-＞is_klass（），"Should be resolved klass"）；
	klassOop k_entry=（klassOop）entry；
	assert（k_entry-＞klass_part（）-＞oop_is_instance（），"Should be instanceKlass"）；
	instanceKlass * ik=（instanceKlass*）k_entry-＞klass_part（）；
	//确保对象所属类型已经经过初始化阶段
	if（ik-＞is_initialized（）＆＆ik-＞can_be_fastpath_allocated（））{
		//取对象长度
		size_t obj_size=ik-＞size_helper（）；
		oop result=NULL；
		//记录是否需要将对象所有字段置零值
		bool need_zero=！ZeroTLAB；
		//是否在TLAB中分配对象
		if（UseTLAB）{
			result=（oop）THREAD-＞tlab（）.allocate（obj_size）；
		}
		if（result==NULL）{
			need_zero=true；
			//直接在eden中分配对象
			retry：
			HeapWord * compare_to=*Universe：heap（）-＞top_addr（）；
			HeapWord * new_top=compare_to+obj_size；
			/*cmpxchg是x86中的CAS指令，这里是一个C++方法，通过CAS方式分配空间，如果并发失败，
			转到retry中重试，直至成功分配为止*/
			if（new_top＜=*Universe：heap（）-＞end_addr（））{
				if（Atomic：cmpxchg_ptr（new_top,Universe：heap（）-＞top_addr（），compare_to）！=compare_to）{
					goto retry；
				}
			result=（oop）compare_to；
			}
		}
		if（result！=NULL）{
			//如果需要，则为对象初始化零值
			if（need_zero）{
				HeapWord * to_zero=（HeapWord*）result+sizeof（oopDesc）/oopSize；
				obj_size-=sizeof（oopDesc）/oopSize；
				if（obj_size＞0）{
					memset（to_zero，0，obj_size * HeapWordSize）；
				}
			}
			//根据是否启用偏向锁来设置对象头信息
			if（UseBiasedLocking）{
				result-＞set_mark（ik-＞prototype_header（））；
			}else{
				result-＞set_mark（markOopDesc：prototype（））；
			}
		result-＞set_klass_gap（0）；
		result-＞set_klass（k_entry）；
		//将对象引用入栈，继续执行下一条指令
		SET_STACK_OBJECT（result，0）；
		UPDATE_PC_AND_TOS_AND_CONTINUE（3，1）；
		}
	}
}
```

### 	二、对象的内存分配和布局

#### 	虚拟内存中对象的存储可以分为三个区域：对象头Header，实例数据Instance Data，对齐补充padding。

​	对象头Header包含两方面的内容：

​		1、存储对象自身运行的时测数据；

​		2、类型指针；

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201019131534204.png" alt="image-20201019131534204" style="zoom: 80%;" />

### 	三、对象的方位定位

​		通过句柄访问对象，如图所示：

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201019131827103.png" alt="image-20201019131827103" style="zoom:80%;" />

​		通过指针访问对象，如图书所示：

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201019132030467.png" alt="image-20201019132030467" style="zoom:80%;" />

## JVM内存分配参数以及实战：OutOfMemoryError异常

### 	一、JVM内存分配参数

#### 	1、堆内存

​		-Xmx：最大对内存

​		-Xms：最小堆内存

​		**注意：jvm会试图将系统的内存尽可能限制在-Xms中，当内存使用量触及-Xms制定大小的时候，就会触发FullGC。因此把-Xms设置为-Xmx时，可以在系统能运行时减少初期的GC次数。**

​		-XX:MinHeapFreeRatio：设置对空间最小空闲比例，当对空间内存小于这个数值的时候，jvm就会自动扩展；

​		-XX:MaxHeapFreeRatio：设置堆空间的最大空闲比例，当对空间内存大于这个数值的时候，jvm就会压缩空间，得到一个最小堆。

#### 	2、新生代

​		-Xmn：设置新生代；

​			-XX:NewSize：设置新生代的初始化大小；-XX:MaxNewSize：设置新生代最大值；

​			**注意：设置不同的-XX:NewSize和-XX:MaxNewSize值会导致内存震荡**

​		新生代一般会设置为整个对空间的1/4到1/3之间。

#### 	3、持久代

​		-XX:PermSize：设置持久代的初始化大小

​		-XX:MinPremSize：设置持久代的最大值

​		**注意：持久代的大小直接决定了系统更新可以支持多少个类定义和多少个常量**

#### 	4、线程栈

​		-Xss：设置线程的大小

#### 	5、堆比例分配

​		-XX:SurvivorRatio：设置新生代中eden和S0的比例关系

​		-XX:TargetSurvivorRation：设置survivor区域的可使用率。当Survivor区域的使用率达到这个数值的时候，会将送入到年老代。

### 	二、实战：OutOfMemoryError异常

### 	1、Java堆溢出

​	**问题：对象数量到达最大堆的容量限制后就会产生内存溢出异常。**

​	异常代码：

```java
java.lang.OutOfMemoryError：Java heap space
Dumping heap to java_pid3404.hprof……
Heap dump file created[22045981 bytes in 0.663 secs]
```

​	解决办法：

​	若是内存泄漏，通过工具检查对象到GC Root的引用链，进一步找出为什么这些对象不能被GC回收；若是不存在内存泄漏的问题，那么我就通过设置-Xmx：最大对内存和-Xms：最小堆内存类增加Java堆空间；最后从代码层面上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。

### 	2、虚拟机栈和本地方法栈溢出

**问题：1、如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常。2、如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常。**

异常代码：

```java
stack length：2402
Exception in thread"main"java.lang.StackOverflowError
at org.fenixsoft.oom.VMStackSOF.leak（VMStackSOF.java：20）
at org.fenixsoft.oom.VMStackSOF.leak（VMStackSOF.java：21）
at org.fenixsoft.oom.VMStackSOF.leak（VMStackSOF.java：21）
……
```

注意问题：

​	**1、使用-Xss参数减少栈内存容量。结果：抛出StackOverflowError异常，异常出现时输出的堆栈深度相应缩小。定义了大量的本地变量，增大此方法帧中本地变量表的长度。结果：抛出StackOverflowError异常时输出的堆栈深度相应缩小。结果表明：在单个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。**

**2、如果不仅仅限制于单线程，那么在不断的创建线程的过程中，可能会产生内存溢出的异常，但是这样产生的内存溢出异常与栈空间是否足够大并不存在任何联系，也就是说在这种情况下，为每个线程的栈分配的内存越大，反而越容易产生内存溢出异常。**

造成以上问题原因：

**操作系统分配给每个进程的内存是有限制的，譬如32位的Windows限制为2GB。虚拟机提供了参数来控制Java堆和方法区的这两部分内存的最大值。剩余的内存为2GB（操作系统限制）减去Xmx（最大堆容量），再减MaxPermSize（最大方法区容量），程序计数器消耗内存很小，可以忽略掉。如果虚拟机进程本身耗费的内存不计算在内，剩下的内存就由虚拟机栈和本地方法栈“瓜分”了。每个线程分配到的栈容量越大，可以建立的线程数量自然就越少，建立线程时就越容易把剩下的内存耗尽。**

### 	3、方法区和运行时常量池溢出

​	CGLib使方法区出现内存溢出异常：

```java
public class test {
    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }
    static class OOMObject {
    }
}
```

​	结果：

```java
Caused by：java.lang.OutOfMemoryError：PermGen space
at java.lang.ClassLoader.defineClass1（Native Method）
at java.lang.ClassLoader.defineClassCond（ClassLoader.java：632）
at java.lang.ClassLoader.defineClass（ClassLoader.java：616）
……8 more
```

​	解决方案：

​	方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾收集器回收掉，判定条件是比较苛刻的。在经常动态生成大量Class的应用中，需要特别注意类的回收状况。

### 	4、本机直接内存溢出

​	异常：

```java
Exception in thread"main"java.lang.OutOfMemoryError
at sun.misc.Unsafe.allocateMemory（Native Method）
at org.fenixsoft.oom.DMOOM.main（DMOOM.java：20）
```

​	解决方案：

由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果读者发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的原因。

## 垃圾收集器与内存分配策略--对象已死吗？

### 	通过对象被引用的方式的了解，就能知道在对象不被引用或者在长时间没引用的情况下，GC应该怎么处理。

#### 1、引用计数算法

​	给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。

​	但是在java虚拟机中没有引入这样的管理方式，因为在Java中它很难解决对象之间相互循环引用的问题。

#### 2、可达性分析计算法

​	通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference  Chain），当一个对象到GC  Roots没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。

![image-20201023163355577](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201023163355577.png)

#### 		可作为GC Roots的对象包括下面几种：

​			虚拟机栈（栈帧中的本地变量表）中引用的对象。
​			方法区中类静态属性引用的对象。
​			方法区中常量引用的对象。
​			本地方法栈中JNI（即一般说的Native方法）引用的对象。

#### 3、引用

​	Java中的定义：如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用。也就是说当内存空间还足够时，则能保留在内存之中；如果内存空间在进行垃圾收集后还是非常紧张，则可以抛弃这些对象。

​	在JDK1.2以后，Java对引用进行分类：强引用（Strong Reference）、软引用（Soft  Reference）、弱引用（Weak  Reference）、虚引用（Phantom Reference）4种，这4种引用强度依次逐渐减弱。

### 	强引用（Strong Reference）

```java
Object obj = new Object();//对象强引用
```

​	只要强应用存在，那么垃圾收集器就永远不会回收这些被应用的对象；如果想要终断强引用和某个对象之间的关联，那么直接将对象赋值为空即可，obj=null。

### 	软引用（Soft  Reference）

```java
java.lang.ref.SoftReference;//软引用
SoftReference softReference = new SoftReference(new WxAdminInfoDo());
```

​	描述一些有用但并非必需的对象；在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。

### 	弱引用（Weak  Reference）

```java
java.lang.ref.WeakReference;//弱引用
WeakReference weakReference = new WeakReference(new WxPlanPhotoDo());
```

​	描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。

### 	虚引用（Phantom Reference）

```java
java.lang.ref.PhantomReference;//虚引用
```

​	一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。

## 垃圾回收算法

#### 	一、标记删除算法

​	在讨论这个问题时，我们得从GC的基础算法说起，在GC中主要使用的是标记删除算法，所谓的标记删除算法就是在程序运行过程中，如果可以用的内存被耗尽的时候，GC线程就会被处罚并将程序暂停，随后将依旧存活的对象标记一遍，最终将对中没有被标记的对象全部清除，接下来恢复程序运行。如图所示。

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026094443559.png" alt="image-20201026094443559" style="zoom: 50%;" />

​	在标记删除算法中主要使用的是可达性分析算法，遍历所有的GC ROOT，然后将所有GC ROOT可达对象标记为存活对象；清除的没有被标记的对象。这里主要筛选的条件是此对象是否有必要执行finalize（）方法，要是对象没有覆盖finalize（）方法，或者finalize（）方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。如图所示。

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026093756331.png" alt="image-20201026093756331" style="zoom:80%;" />

​	标记删除算法的缺点：

​	1、首先，**它的缺点就是效率比较低（递归与全堆对象遍历），而且在进行GC的时候，需要停止应用程序，这会导致用户体验非常差劲**，尤其对于交互式的应用程序来说简直是无法接受。

​	2、**这种方式清理出来的空闲内存是不连续的**，这点不难理解，我们的死亡对象都是随即的出现在内存的各个角落的，现在把它们清除之后，内存的布局自然会乱七八糟。而为了应付这一点，JVM就不得不维持一个内存的空闲列表，这又是一种开销。而且在分配数组对象的时候，寻找连续的内存空间会不太好找。

#### 	二、复制删除算法

​	所谓的复制删除算法就是将内存中按照相同的区域划分为大小相等的两块区域，每次只使用其其中的一块，当着一块使用完了，那么就将存活的对象移动到另一块上面，然后在把自己的内存清理一遍，这样每次清理的只有一半的内存空间。在不考虑内存碎片的情况下，只需要移动堆顶指针就能完成内存分配，运行效率高，但是这样的代价就是缩小了原有的内存。如图所示。

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026095555672.png" alt="image-20201026095555672" style="zoom:50%;" />

​	现代商业的虚拟机都是采用这种复制删除算法来回收新生代的。新生代中的对象98%是“朝生夕死”的，所以并不需要按照1:1的比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。HotSpot虚拟机默认Eden和Survivor的大小比例是8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%（80%+10%），只有10%的内存会被“浪费”。当然，98%的对象可回收只是一般场景下的数据，我们没有办法保证每次回收都只有不多于10%的对象存活，当Survivor空间不够用时，需要依赖其他内存（这里
指老年代）进行分配担保（Handle Promotion）。如图所示：

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026100345557.png" alt="image-20201026100345557" style="zoom: 67%;" />

#### 	三、标记整理算法

​	标记完成对象以后将所有存活的对象整理至内存的另一端，然后清除边界意外的空间。

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026101637672.png" alt="image-20201026101637672" style="zoom:50%;" />

#### 	四、增量算法

​	如果一次性将所有的垃圾进行清除，那么会造成系统长时间的顿卡。解决方案：让垃圾收集器线程和应用系统线程交替进行。

​	缺点：线程切换和上下文的消耗会使得垃圾回收的成本上升，造成系统吞吐量降低。

#### 	五、分代收集算法

​	思想：将系统根据对象的特点分类，根据每个区域的特点使用不同的回收算法，以提高效率；

​	新生代：使用复制删除算法；90%的会被回收； 

​	年来代：使用标记整理算法；大部分是存活对象；

## 垃圾收集器

<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026102648914.png" alt="image-20201026102648914" style="zoom: 67%;" />

### 	一、Serial收集器

​		<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026131823376.png" alt="image-20201026131823376" style="zoom: 80%;" />

​		新生代串行收集器；

​		单线程收集器，在工作中的时候必须暂停所有工作线程，即Stop The Worlld；

​		优点：简单、搞笑、处理逻辑简单、没有线程切换的开销；

​		缺点：程序顿卡

### 	二、ParNew收集器

​		<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026132421371.png" alt="image-20201026132421371" style="zoom:80%;" />

​		ParNew收集器其实就是Serial收集器的多线程版本；

​		新生代并行收集器；

### 	三、Parallel Scavenger收集器

​	目标则是达到一个可控制的吞吐量（Throughput）；（所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间），虚拟机总共运行了100分钟，其中垃圾收集花掉1分钟，那吞吐量就是99%。）

​	使用复制删除算法并行多线程收集器；

### 	四、Serial Old收集器

​	<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026133020486.png" alt="image-20201026133020486" style="zoom:80%;" />

​	老年代单线程串行收集器；

### 	五、Parallel Old收集器

​	<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026133205446.png" alt="image-20201026133205446" style="zoom:80%;" />

​	老年代多线程并行收集器；

### 	六、CMS收集器

​	<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026133808224.png" alt="image-20201026133808224" style="zoom:80%;" />

​	一种以获取最短回收停顿时间为目标的收集器；希望系统停顿时间最短，用以给用户最好的体验；

​	基于标记删除删除算法实现；

​	多线程并行回收；

​	主要步骤，如上图所示：

​		初始标记（CMS initial mark）

​		并发标记（CMS concurrent mark）

​		重新标记（CMS remark）

​		并发清除（CMS concurrent sweep）

​	缺点：1、初始化标记和重新标记要独立占用系统资源；2、并发标记和并发删除和用户线程一起，比较消耗资源以及线程上下文切换时的时间消耗；

### 	七、G1收集器

​	<img src="C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026134719163.png" alt="image-20201026134719163" style="zoom:80%;" />

​	面向服务端应用的垃圾收集器；

​	步骤如上图所示：

​	初始标记（Initial Marking）
​	并发标记（Concurrent Marking）
​	最终标记（Final Marking）
​	筛选回收（Live Data Counting and Evacuation）

​	优点：

​		1、并行与并发：G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU（CPU或者CPU核心）来缩短Stop-The-World停顿的时间；

​		2、分代收集

​		3、空间整合：整体来看是基于“标记—整理”算法实现的收集器，从局部（两个Region之间）上来看是基于“复制”算法实现的；

​		4、可预测的停顿

## 内存分配与回收策略

### 	Java技术体系中内存管理最终规化为两个问题：1、给对象分配内存；2、回收给对象分配的内存；

#### 	一、对象优先在Eden分配

#### 	二、大对象直接进入老年代

#### 	三、长期存活的对象将进入老年代

#### 	四、动态对象年龄判定

#### 	五、空间分配担保

## 虚拟机性能监控与故障处理工具

### 	一、JPS：虚拟机进程状况工具

```sh
##主要参数：
##-q 只输出进程ID
##-m 用于输出传递给java进程（main函数）的参数
##-l 用于输出主函数的完整路径
jps[options][hostid]
```

​	列出Java的进程；

```shell
jps -l
```

### 	二、jstat：虚拟机统计信息监视工具

```shell
## 主要参数：
## -t ：代表时间戳
## -h<lines>：即-h跟数字，代表隔几行显示标题
## vmid ：代表vm进程id
## interval：代表监控间隔时间段，默认毫秒做单位
## count：代表取数次数
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```

​	![img](C:\Users\Administrator\AppData\Local\YNote\data\cxg1207@126.com\642707b36d6645d7a22b5f85b7307ef1\clipboard.png)

**堆内存 = 年轻代 + 年老代 + 永久代 + 元数据区 **

**年轻代 = Eden区 + 两个Survivor区（From和To）**

```shell
jstat -gcutil -t -h5 pid 1000 100 ##表示每1000毫秒收集一次jvm内存和gc信息，共收集100次，每隔5行显示一次标题，且标题行带时间戳
```

![image-20201026141258242](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026141258242.png)

```shell
jstat -gc pid ##垃圾回收统计
##查询后参数详解：
- S0C：第一个幸存区的大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- OC：老年代大小
- OU：老年代使用大小
- MC：方法区大小
- MU：方法区使用大小
- CCSC:压缩类空间大小
- CCSU:压缩类空间使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间
```

```shell
jstat -gcutil pid ##总结垃圾回收统计
##查询后参数详解：
S0：幸存1区当前使用比例
S1：幸存2区当前使用比例
E：伊甸园区使用比例
O：老年代使用比例
M：元数据区使用比例
CCS：压缩使用比例
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

```shell
jstat -gccapacity pid ##堆内存统计
#查询后参数详解：
NGCMN：新生代最小容量
NGCMX：新生代最大容量
NGC：当前新生代容量
S0C：第一个幸存区大小
S1C：第二个幸存区的大小
EC：伊甸园区的大小
OGCMN：老年代最小容量
OGCMX：老年代最大容量
OGC：当前老年代大小
OC:当前老年代大小
MCMN:最小元数据容量
MCMX：最大元数据容量
MC：当前元数据空间大小
CCSMN：最小压缩类空间大小
CCSMX：最大压缩类空间大小
CCSC：当前压缩类空间大小
YGC：年轻代gc次数
FGC：老年代GC次数
```

```shell
jstat -gcmetacapacity pid ##元数据空间统计
##查询后参数详解：
 MCMN:最小元数据容量
MCMX：最大元数据容量
MC：当前元数据空间大小
CCSMN：最小压缩类空间大小
CCSMX：最大压缩类空间大小
CCSC：当前压缩类空间大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

```shell
jstat -gcnewcapacity pid ##新生代内存空间统计
## 查询后参数详解：
NGCMN：新生代最小容量
NGCMX：新生代最大容量
NGC：当前新生代容量
S0CMX：最大幸存1区大小
S0C：当前幸存1区大小
S1CMX：最大幸存2区大小
S1C：当前幸存2区大小
ECMX：最大伊甸园区大小
EC：当前伊甸园区大小
YGC：年轻代垃圾回收次数
FGC：老年代回收次数
```

```shell
jstat -gcoldcapacity pid ##老年代内存空间统计
##查询后参数详解：
OGCMN：老年代最小容量
OGCMX：老年代最大容量
OGC：当前老年代大小
OC：老年代大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

```shell
jstat -gcnew pid ##新生代垃圾回收统计
##查询后参数详解：
- S0C：第一个幸存区大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- TT:对象在新生代存活的次数
- MTT:对象在新生代存活的最大次数
- DSS:期望的幸存区大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间
```

### 	三、jinfo：Java配置信息工具

```shell
jinfo [option] pid ##用来查询正在运行的java应用程序的扩展参数
```

![image-20201026142722862](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026142722862.png)

### 	四、jmap：Java内存映像工具

```shell
jmap [option] vmid ##生成Java应用程序的堆快照和对象的统计信息
```

```shell
jmap -histo pid >d:\s.txt
```

```shell
jmap -dump:fromat=b,file=c:heap.hprof pid ##获取当前java的堆快照
```

### 	五、jhat：虚拟机堆转储快照分析工具

```shell
jhat eclipse.bin
```

### 	六、jstack：Java堆栈跟踪工具

```shell
jstack [option] vmid
```

```shell
jstack -l pid ##除堆栈信息外，显示关于锁的附加信息
```

![image-20201026143707762](C:\Users\Administrator\Desktop\Learn Plan_A Java\java虚拟机 学习笔记\img\image-20201026143707762.png)

### 	七、HSDIS：JIT生成代码反汇编

### 	八、JDK的可视化工具

​		1、JConsole：Java监视与管理控制台--java自带

​		2、VisualVM：多合一故障处理工具--第三方

