#JVM内存垃圾回收

##1.jvm垃圾收集算法

- 1.引用计算算法（jdk1.2之前）

		每个对象有一个引用计数属性，新增一个引用时计数加1，引用释放时计数减1，计数为0时可以回收。此方法简单，无法解决对象相互循环引用的问题。还有一个问题是如何解决精准计数。
- 2.根搜索算法

		从GC Roots开始向下搜索
，搜索所走过的路径成为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的，不可达对象

		在java语言中，GC Roots包括:
		虚拟机栈中引用的对象(最常用)
		方法区中类静态熟性实体引用的对象(很少用)
		方法区中常量引用的对象(很少用)
		本地方法栈中JNI引用的对象

##2.jvm垃圾回收算法

- 1、复制算法(copying)

		1.复制算法采用从根集合扫描，并将存活对象复制到一块新的，没有使用过的空间中，这种算法当控件存活的对象比较少时，极为高效，但是带来的成本是需要一块内存交换空间，用于进行对象的移动
		2.此算法用于新生代内存回收，从E区回收到S0或者S1	
- 2、标记清除算法(Mark-Sweep)

		1.标记-清除算法采用从根集合进行扫描，对存活的对象进行标记，标记完毕后，再扫描整个空间中未被标记的对象，进行回收，如何所示；
		2.标记-清除算法不需要进行对象的移动，并且仅对不存活的对象进行处理，在存活对象比较多的情况下极为高效（使用于老年代），但由于标记-清除算法直接回收不存活的对象，因

此会造成内存碎片！
- 3、标记整理算法(Mark-Compac)

		标记-整理算法采用标记-清除算法一样的方式进行标记，但是清除的时候不同，在回收不存活的对象占用的空间后，会将所有的存活对象往左端空闲空间移动，并更新对应的指针。标记-整理算法在标记-清除算法的基础上，又进行了对象的移动，因此成本很高，但却解决了内存碎片的问题。

##3.相关的名词解释

- 1 串行回收

		gc单线程内存回收，会暂停所有用户线程

- 2 并行回收

 		收集是指多个GC线程并行工作，但此时用户线程是暂停的；所以serial是串行的，parallel收集器是并行的，而CMS收集器是并发的
		
- 3 并发回收

		是指用户线程与GC线程是同时执行(不一定是并行，可能交替，但总体是在同时执行的)，不需要停顿用户线程(其实在CMS中用户线程还是需要停顿的，只是非常短，GC线程在另一个CPU上执行)

##3.1 serial回收器(串行回收器)

	1.是一个单线程的收集器，只能使用一个CPU或一条线程去完成垃圾收集；在进行垃圾收集时，必须暂停所有其他的工作线程，直到收集完成。
	2.缺点：stop the world
	3.优势：简单。但对于CPU的情况，由于没有多线程的互相开销，反而可以更高效，是client模式下默认的新生代收集器
		
	新生代Serial回收器
	1.-XX:+UseSerialGC来开启
	Serial New + Serial Old的收集器组合进行内存回收
	2.使用复制算法
	3.独占式的垃圾回收
		一个线程进行GC、串行。其他的工作线程暂停
		
	老年代Serial回收器
	1.-XX:+UseSerialGC来开启
	Serial New + Serial Old的收集器组合进行内存回收
	2.使用标记-压缩算法
	3.串行的、独占式的垃圾回收器
	因为内存比较大的原因，回收比新生代慢

##3.2 ParNew 回收器(并行回收器)
	并行回收器也是独占式的回收器，在收集过程中，应用程序会全部暂停。但由于并行回收器使用多线程进行垃圾回收，因此，在并发能力比较强的CPU上，它产生的停顿时间要短于串行回收器，而在单CPU或者并发能力较弱的系统中，并行回收器的效果不会比串行回收器好，由于多线程的压力，它的实际表现很可能比串行回收器差	

	新生代ParNew回收器
	1.-XX:+UseParNewGC开启
		新生代使用并行回收收集器，老年代使用串行收集器
	2.-XX:ParallelGCThreads指导线程数
		默认最好与CPU数量相当，避免过多的线程数影响垃圾回性能
	3.使用复制算法
	4.并行的、独占式的垃圾回收器

	新生代ParNew Scavenge回收器
	1.吞吐量优先回收器
	关注CPU吞吐量，即运行用户代码的时间/总时间，比如:JVM运行100分钟，其中运行用户代码99分钟，垃圾回收1分钟，则吞吐量99%，这种收集器能最高效率的利用CPU，适合后台运算
	2.-XX:+UseParallelGC开启
	使用Parallel Scavenge+Serial Old收集器组合回收垃圾，这也是在Server模式下的默认值
	3.-XX:GCTimeRatio
	来设置用户执行时间占总时间的比例，默认是99，即1%的时间用来垃圾回收
	4.-XX:MaxGCPauseMilis
	设置GC最大的停顿时间
	5.使用复制算法

	老生代ParNew Old回收器
	1.-XX:+UseParallelOldGC开启
	使用Parallel Scavenge+Serial Old组合收集器进行收集
	2.使用标记整理算法
	3.并行的、独占时的垃圾回收器

##3.3 CMS 回收器(并发标记清除)

	CMS(并发标记清除)回收器
	运行过程分为4个阶段:
	 初始阶段(CMS initial mark)：值标记GC Roots能直接关联的对象
	 并发标记(CMS concurrent mark)：进行 GC Roots Tracing的过程
	 重新标记(CMS remark)：修正并发标记期间因用户继续运行而导致标记发生改变的那一部分对象的标记
	 并发清除(CMS concurrent sweep)：
	期中标记和重新标记两个阶段仍然需要stop-the-world，整个过程中耗时最长的并发标记和并发清除过程中收集器都可以和用户线程一起工作

- 1、标记-清除算法

		同时它是一个使用多线程并发回收的垃圾回收器
- 2、-XX:ParallelCMSThreads

		手工设定CMS的线程数量，CMS默认启动的线程数是(ParallelGCThreads+3)/4
- 3、-XX:+UseConcMarkSweepGC开启

		使用ParNew+CMS+ Serial Old的收集器组合进行内存回收，Serial Old作为CMS出现"Concurrent Mode Failure"失败后的后备收集器使用
- 4、-XX:CMSInitiatingOccupancyFration

		设置CMS收集器在老年代空间被使用多少后出发垃圾收集，默认值是68%，仅在CMS收集器有效，-XX:CMSInitiatingOccupancyFration=70
- 5、-XX:+UseCMSCompactAtFullCollection

		由于CMS收集器会产生碎片，此参数设置在垃圾回收器后是否需要一次内存碎片整理过程，仅在CMS收集器时有效。
- 6、-XX:+CMSFullGCBeforeCompaction

		设置CMS收集器在进行若干次垃圾收集后再进行一次内存碎片整理过程，通常与UseCMSCompactAtFullCollection参数一起使用
- 7、-XX:CMSInitiatingPermOccupancyFration（你不建议设置）

		设置Perm Gen使用到达多少比率时触发，默认92s


##3.4 GC性能指标

- 吞吐量  应用花在非GC上的时间百分比
- GC负荷  与吞吐量相反，指应用花在GC上的时间比
- 暂停时间 应用花在GC stop-the-world的时间
- GC频率  顾名思义
- 反应速度 从一个对象变成垃圾到这个对象被回收的时间
- 一个交互式的应用要求暂停时间越少越好，然而，一个非交互性的应用，当然是希望GC负荷越低越好
- 一个实时系统对暂停时间和GC负荷的要求，都是越低越好


##4.内存容量配置原则

###4.1 年轻代大小选择

	响应时间优先的应用：尽可能设大，直到接近系统的最低响应时间设置(根据实际情况选择)，在此种情况下，年轻代收集发送的频率也是最小的。同时，减少到达年老代的对象

	吞吐量优先的应用：尽可能的设置大，可能到达Gbit的程度，因为对响应时间没有要求，垃圾回收可以并行进行，一般适合8CPU以上的应用

	避免设置过小，当新声代设置过小时会导致：1YGC次数更加频繁 2.可能导致YGC对象直接进入旧生代，会出发fullGC

###4.2 年老代大小选择

	响应时间优先的应用：年轻代使用并发收集器，所以其大小需要小心设置，一般要考虑并发会话频率和会话持续时间等一些参数，如果堆设置小了。可以造成会话碎片，高回收频率以及应用暂停而使用传统的标记清除的方式;如果堆大了，则需要较长的收集时间，最优化的方案，一般需要参考一下数据获得：

    并发垃圾收集信息、持久代并发收集次数、传统GC信息、花在年轻代和年老代回收的时间比例
    吞吐量优先的应用：一般吞吐量的应用都有一个很大的年轻代和一个较小的老年代。原因是：这样可以尽可能回收掉大部分短期对象，减少中期的对象，而老年代尽存放长期存活的对象
