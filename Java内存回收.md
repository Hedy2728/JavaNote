# Java的内存回收

[推荐文章](https://mp.weixin.qq.com/s/MeDsGjpBGb4XX1GIuPF60Q)

## 一、什么时候垃圾回收

1、当应用程序分配新的对象，GC的代的预算大小已经达到阈值，比如GC的第0代已满

2、代码主动显式调用System.GC.Collect()

3、其他特殊情况，比如，windows报告内存不足、CLR卸载AppDomain、CLR关闭，甚至某些极端情况下系统参数设置改变也可能导致GC回收

### 1、对象已死？

当对象死去的时候进行垃圾回收，即对象失去引用的时候进行垃圾回收。**如何判断对象已经死去呢？**

+ 引用计数算法：在对象中添加一个引用计数器，每当有一个地方引用时就加一，引用失效时就减一，为零的时候就进行垃圾回收。此方法的**缺点**是当有两个对象互相引用时，实际上这两个对象已经不再被访问，由于循环引用的存在，导致引用计数器恒不为零。
+ 可达性分析算法：通过一系列的被称为GC Roots的对象作为起始点，从这些起始点往下搜索，当一个对象到GC Roots没有任何引用链时，即对象不可达，则此对象可以进行回收

GC Roots包括以下几种：

+ 虚拟机栈（栈帧中的本地变量表）中引用的对象
+ 方法区中类静态属性引用的对象
+ 方法区中常量引用的对象（final常量值）
+ 本地方法中JNI引用的对象（Native方法）

### 2、引用

引用包含以下几种：

+ 强引用
+ 软引用
+ 弱引用
+ 虚引用

### 3、对象不一定非死不可

真正宣告一个对象死亡至少经历两次标记过程：如果第一次找不到GC Roots，则判断是否需要进行finalize()方法，如果对象已经执行过finalize()或者finalize()已经被覆盖，则不用执行finalize()。对象在执行finalize()期间如果与引用链上的对象挂上钩则第二次标记时会将其清理出即将回收的集合，否则将被回收。

## 二、垃圾回收的对象是什么

所谓内存回收，当然是回收内存。Java运行时数据区主要分为以下几部分：

+ 程序计数器：当前线程所执行字节码的行号指示器，唯一不存在OOM的区域

+ 虚拟机栈：每个方法在执行的同时会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。为字节码服务，即Java方法

  > Java栈中存放的是一个个的栈帧，每个栈帧对应一个被调用的方法，在栈帧中包括局部变量表(Local Variables)、操作数栈(Operand Stack)、指向当前方法所属的类的运行时常量池（运行时常量池的概念在方法区部分会谈到）的引用(Reference to runtime constant pool)、方法返回地址(Return Address)和一些额外的附加信息。当线程执行一个方法时，就会随之创建一个对应的栈帧，并将建立的栈帧压栈。当方法执行完毕之后，便会将栈帧出栈。因此可知，线程当前执行的方法所对应的栈帧必定位于Java栈的顶部。讲到这里，大家就应该会明白为什么 在 使用 递归方法的时候容易导致栈内存溢出的现象了以及为什么栈区的空间不用程序员去管理了（当然在Java中，程序员基本不用关系到内存分配和释放的事情，因为Java有自己的垃圾回收机制），这部分空间的分配和释放都是由系统自动实施的。对于所有的程序设计语言来说，栈这部分空间对程序员来说是不透明的。

+ 本地方法栈：为Native方法服务，与虚拟机栈功能类似

+ Java堆：在虚拟机启动时创建，存放对象实例，所有对象实例和数组都要在堆上分配（现在没有这么绝对）

  > 逃逸分析：通过逃逸分析来决定某些实例或者变量是否要在堆中进行分配，如果开启了逃逸分析，即可将这些变量直接在栈上进行分配，而非堆上进行分配。这些变量的指针可以被全局所引用，或者被其它线程所引用。

+ 方法区（永生代）：存放已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码

+ 运行时常量池（包括在方法区中）：存放编译期生成的各种字面量与符号引用

1-3属于线程私有，4-6是线程共享数据区域，垃圾收集器关注的是数据共享部分的内存区域。**Java堆是内存回收的主要对象**

方法区的回收性价比不高，主要包括两部分内容：废弃常量和无用的类。废弃常量直接看是否有引用，而无用的类包括以下三种：

+ 该类所有实例都已经被回收，Java堆中没有任何相关实例
+ 加载该类的ClassLoader已被回收
+ 该类对应的java.lang.Class对象没有在任何地方引用，无法在任何地方通过反射访问该类的方法

## 三、用什么垃圾回收算法

### 1、标记-清除算法

标记出需要回收的对象，标记完成后统一收集。

两个缺点：

+ 效率不高：标记和清除效率不高
+ 空间利用不强：标记清除后产生大量内存碎片，不利于内存分配

### 2、复制算法

为了解决效率问题，将可用内存划分为大小相同的两块，每次只用其中一块。一块用完之后再将对象复制到另外一块上，清除内存只需要清理一半。但是内存利用率不高。

目前采用此算法回收新生代，内存空间化为8份Eden，2份Survivor。当Survivor不够时，需要其他内存（老年代）进行分配担保

现在我们应该明白把新生代设置成 Eden, S0，S1区或者给对象设置年龄阈值或者默认把新生代与老年代的空间大小设置成 1:2 都是为了**尽可能地避免对象过早地进入老年代，尽可能晚地触发 Full GC**。想想新生代如果只设置 Eden 会发生什么，后果就是每经过一次 Minor GC，存活对象会过早地进入老年代，那么老年代很快就会装满，很快会触发 Full GC，而对象其实在经过两三次的 Minor GC 后大部分都会消亡，所以有了 S0,S1的缓冲，只有少数的对象会进入老年代，老年代大小也就不会这么快地增长，也就避免了过早地触发 Full GC。

由于 Full GC（或Minor GC） 会影响性能，所以我们要在一个合适的时间点发起 GC，这个时间点被称为 Safe Point，这个时间点的选定既不能太少以让 GC 时间太长导致程序过长时间卡顿，也不能过于频繁以至于过分增大运行时的负荷。一般当线程在这个时间点上状态是可以确定的，如确定 GC Root 的信息等，可以使 JVM 开始安全地 GC。Safe Point 主要指的是以下特定位置：

- 循环的末尾
- 方法返回前
- 调用方法的 call 之后
- 抛出异常的位置 另外需要注意的是由于新生代的特点（大部分对象经过 Minor GC后会消亡）， Minor GC 用的是复制算法，而在老生代由于对象比较多，占用的空间较大，使用复制算法会有较大开销（复制算法在对象存活率较高时要进行多次复制操作，同时浪费一半空间）所以根据老生代特点，在老年代进行的 GC 一般采用的是标记整理法来进行回收。

### 3、标记整理算法

根据老年代提出此算法，标记后将存活对象都向一端移动，然后清理掉边界以外的内存

### 4、分代收集算法

根据不同年代的特点采用最适当的收集算法。

## 四、用什么垃圾回收器

### 1、Serial收集器

单线程收集器，他在进行垃圾收集时，其他线程必须停止工作，Stop The World，适用于单个CPU的工作环境，没有线程交互的开销。对于运行在Client模式下的VM是很好的选择

### 2、ParNew收集器

Serial收集器的多线程版本，也是需要Stop The World，除了Serial之外，只有ParNew能够配合CMS工作

### 3、Parallel Scavenge收集器

新生代收集器，使用复制算法的收集器，关注点在于达到一个可控制的吞吐量，停顿时间越短越适合需要与用户交互的程序，高吞吐量则可以更高效地利用CPU时间，尽快完成程序的运算任务，适合在后台运算而不需要交互太多的任务。GC的停顿时间缩短是牺牲吞吐量和新生代空间而换取。自适应调节策略是他与ParNew收集器的重要区别

### 4、Serial Old收集器

Serial收集器的老年代版本，Client模式下的VM使用，两个用途：一个是在JDK1.5之前的版本中与Parallel Scavenge收集器搭配使用，另一个用途是作为CMS收集器的备选方案

### 5、Parallel Old收集器

Parallel Scavenge收集器的老年代版本，使用多线程标记和标记整理算法。由于CMS无法在JDK1.5之后与Parallel Scavenge搭配使用，所以有了Parallel Old。

### 6、CMS收集器

关注点为尽可能缩短垃圾收集时用户线程停顿的时间，基于标记清除算法实现，分为以下四步：

+ 初始标记（Stop The World）：标记GC Roots能够关联到的对象
+ 并发标记
+ 重新标记（Stop The World）：修正并发标记期间程序继续运行而导致的对象标记变动
+ 并发清除

同时，CMS也有缺点：

+ 对CPU资源敏感：在并发阶段虽然不会导致用户线程停顿，但是因为占用一部分系统资源导致应用程序变慢，总吞吐量降低
+ 无法处理浮动垃圾：CMS收集器与用户线程并发进行，从而用户线程会产生新的垃圾，只能等待下一个GC清理。而且CMS需要留出一部分内存提供并发收集时的程序运行使用
+ CMS基于标记清除实现，会产生大量空间碎片，为了解决此问题，CMS提供内存整理功能

### 7、G1收集器

G1有计划的在Java堆中避免全区域的垃圾收集，跟踪每个Region的垃圾价值大小，在后台维护一个优先列表。优点如下：

+ 并行与并发：多CPU减少Stop The World的时间
+ 分代收集：采用不同的方式对待存活时间不同的对象
+ 空间整合：G1将Java堆划分为多个大小相等的Region，从局部看基于复制算法实现，收集后提供规整的可用内存，运作期间不产生内存空间碎片
+ 可预测的停顿：使用者可明确指定在某段时间内消耗在垃圾收集的时间不得超过多长时间

使用G1收集器的时候，Java堆的内存布局就和其他的收集器有很大的差别，G1将Java堆分为很多的Region，虽然还存在着老年代与新生代的概念，但是它们都是一部分Region的合集。而且G1收集器的时间停顿是可以预测的（使用者可以指定M秒的时间段内消耗在垃圾收集上的时间少于N秒），可以有计划的避免在Java堆中进行全区域的垃圾收集。G1跟踪各个Region中垃圾堆的价值的大小和回收的代价，在后台维护一个优先列表，每次根据允许的收集时间，优先来回收价值最大的Region。这种方法可以有很高的垃圾回收效率。
主要步骤有：

+ 初始标记（仅标记与GC Root直接关联的对象）
+ 并发标记（可达性分析，标记活着的对象）
+ 最终标记（修正并发标记时用户线程继续执行产生变动的部分）虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记只要将其数据合并到Remember Set中。（这阶段需要线程停顿，但是可以并行执行）
+ 筛选回收（筛选回收就是维护前面说的那个优先队列，然后根据用户期望的GC停顿时间来进行回收计划）

## 五、内存分配策略

1、对象优先在Eden区分配，内存不够则Minor GC

2、大对象进入老年代

3、长期存活对象进行老年代：有个Age计数器，或者Survivor空间中相同年龄的对象总大小大于Survivor空间的一半，则Age>=该年龄的进入老年代

4、**空间分配担保**：在发生Minor GC之前，VM会检查老年代最大连续空间是否大于新生代所有对象总空间，否则不安全。有可能存在新生代存活率比较高的情况，此时需要老年代做担保，一般取之前每一次回收晋升老年代的平均值作为经验值，与老年代的剩余空间比较，看是Minor GC或是Full GC。

在发生Minor GC之前，虚拟机会检查老年代最大可用连续空间是否大于新生代所有对象的总空间，大的话可以保证安全，因为就算所有都进老年代也能容纳。老年代没有新生代可用空间大的话，虚拟机查看HandlePromotionFailure设置值是否允许担保失败。如果允许，继续检查老年代最大可用连续空间是否大于历次晋升到老年代对象的平均大小，大的话就执行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者HandlePromotionFailure设置不允许冒险，就要改为进行一次FullGC。
冒险冒的什么风险？ 新生代使用复制算法，但是为了内存利用率，只使用其中一个Survivor空间作为轮换备份。因此在MinorGC之后仍有大量对象存活，就要老年代进行分配担保，把Survivor无法容纳的对象直接进入老年代。

## 六、结合垃圾回收，平时写代码有什么注意的地方

由于GC的代价很大，平时开发中注意一些良好的编程习惯有可能对GC有积极正面的影响，否则有可能产生不良效果。

1、尽量不要new很大的object，大对象（>=85000Byte）直接归为G2代，GC回收算法从来不对大对象堆（LOH）进行内存压缩整理，因为在堆中下移85000字节或更大的内存块会浪费太多CPU时间

2、不要频繁的new生命周期很短object，这样频繁垃圾回收频繁压缩有可能会导致很多内存碎片，可以使用设计良好稳定运行的对象池（ObjectPool）技术来规避这种问题

3、使用更好的编程技巧，比如更好的算法、更优的数据结构、更佳的解决策略等等

## 七、java内存泄露

[java内存泄露](https://blog.csdn.net/mengxpfighting/article/details/82184396)

内存泄漏是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。

长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是因为长生命周期持有它的引用而导致不能被回收，这就是Java中内存泄漏的发生场景。


发生内存泄漏的原因以及处理方式：

1、静态集合类引起内存泄漏

　　像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。

2、当集合里面的对象属性被修改后，再调用remove()方法时不起作用。

3、监听器

　　在释放对象的时候却没有去删除这些监听器，增加了内存泄漏的机会。

4、各种连接

　　比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。

5、内部类和外部模块的引用

　　内部类的引用是比较容易遗忘的一种，而且一旦没释放可能导致一系列的后继类对象没有释放。此外程序员还要小心外部模块不经意的引用，例如程序员A 负责A 模块，调用了B 模块的一个方法如： public void registerMsg(Object b); 这种调用就要非常小心了，传入了一个对象，很可能模块B就保持了对该对象的引用，这时候就需要注意模块B 是否提供相应的操作去除引用。

6、单例模式

　　不正确使用单例模式是引起内存泄漏的一个常见问题，单例对象在初始化后将在JVM的整个生命周期中存在（以静态变量的方式），如果单例对象持有外部的引用，那么这个对象将不能被JVM正常回收，导致内存泄漏。
