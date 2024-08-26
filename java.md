#### 1 JVM内存区域

jvm在执行java程序时将使用的内存划分为若干个数据区域称为**运行时数据区**，正是因为将内存控制权交给jvm虚拟机管理，不需要进行垃圾回收，不容易出现内存泄漏和溢出等问题

1. 程序计数器：线程私有，作为当前线程的执行字节码的行号指示器，用于记录当前线程的执行位置（当线程切换时，jvm知道从哪里开始执行）。不会出现outofmemoryerror。

2. java栈：线程私有，栈由栈帧组成，每个方法的调用都会创建一个栈帧，调用结束栈帧会被弹出，栈帧有**局部变量表、操作数栈、动态链接、返回信息**。**局部变量表存放基本数据类型、引用变量**，操作数栈存放方法执行过程中产生的中间计算结果。动态链接存放其他方法的调用，将符号引用转换为方法直接引用。当方法调用深度超过最大深度会stackoverflowerror，当栈的内存可以动态扩展时，若扩展无法申请到足够内存会outofmerory。

3. 方法区：存放类信息、常量、静态变量、即使编译器编译后的代码缓存等。在jdk8后将方法区变成元数据区（**运行时常量池**）

4. 本地方法栈：线程私有，保存native方法信息。与java栈类似，java栈为java方法服务，本地方法栈为native方法服务。

5. 堆：线程共享，存放对象实例。栈中的引用变量指向堆中地址。是GC垃圾收集器管理的区域，因此还可以分为新生代和老生代（eden、survivor、old）

6. 运行时常量池：在方法区或元数据区，存放类在编译期生成的常量信息包括**字面量**（123、“123”、true、null）和**符号引用**，每个类都有一个常量池，存储该类的常量信息。
   [类加载连接的解析步骤中符号引用替换为直接引用是什么意思_将符号引用替换为直接引用-CSDN博客](https://blog.csdn.net/qq_42374233/article/details/120481259)
   - 符号引用：用一组符号来描述所引用的目标，符号可以是任何字面量，与实际地址无关，如类符号引用、字段符号引用、方法符号引用。当源代码被编译成.class文件的时候，如果A引用了B，在编译阶段A并不知道B在虚拟机中的地址（B也未被加载，在方法区中还没有该类)，因此只能使用一个字符串来表示B的地址，即不用知道虚拟机实际上内存的分配。
   - 直接引用：可以直接定位到目标的地址，如指针或者句柄，同一个符号引用在不同虚拟机翻译出来的直接引用可能不同。当A类被在虚拟机中加载的时候，如果引用到了B，需要将B加载到虚拟机中，并将A中对B的符号引用转化为实际的地址，即直接引用
   
7. 字符串常量池
   为了避免字符串重复创建，减少内存消耗。**jdk1.7之前存放在方法区，之后存放在堆中**，因为GC对于堆的回收效率高。
   
8. **对象的创建过程**
- 类加载检查：遇到new指令时，查看这个指令参数在方法区或元数据区是否有该类的符号引用，并且检查是否进行过加载，如果没有加载，进行**类加载过程**（加载、验证、准备、解析、初始化）。
   
- 分配内存：类加载过程结束后，在堆内存中分配空间，根据GC方式的不同有两种分配方式，**指针碰撞**和**空闲列表**，使用指针碰撞的堆内存是规整的，即用过的内存放到一边，沿着没有分配过的方向移动分界指针对象大小的内存。使用空闲列表的堆内存是不规整的，虚拟机维护一个列表记录空闲的内存，为对象分配足够大的内存。
   
- 初始化零值:为对象中成员变量字段赋零值
   
- 设置对象头:对象头，GC年龄、元数据信息标识等，对象头包含两部分信息，一部分存储对象**自身运行时数据**（GC年龄、锁状态、hashcode等），一部分是**类型指针**（表明该对象是哪个类的实例，如元数据信息）。
   
- 执行init方法:虚拟机创建对象完毕，但是java程序中对象执行init方法，按照程序代码初始化。

#### 2 GC策略

1. java中引用 **直接引用**
   java的hotspot虚拟机使用直接引用来访问java对象，**直接引用将引用变量直接指向对象实例，如果需要获得对象类的信息，需要调用对象头中存储的类型指针**

2. 引用强度  **强软弱虚** 
   **无论是使用引用计数法还是可达性分析，判断对象的应用链，在回收时通过引用强度来决定对象是否能被回收**
   1. 强引用：最普遍的都是强引用，未显示指定的都是强引用。当对象存在强引用是不会被回收
   2. 软引用：SoftReference，当对象只有软引用时，在内存不足触发的GC时会被回收，可以与引用队列联合使用。
   3. 弱引用：WeakReference，当对象只有弱引用时，每次GC都会回收无论内存是否足够，可以与引用队列联合使用。
   4. 虚引用：phantomReferenc，形同虚设（**相当于没有引用**），唯一用处在于对象被垃圾回收前进行一个通知，**跟踪对象被垃圾回收的活动**，必须与引用队列联合使用。在垃圾回收前，发现有虚引用则将虚引用加入到相关的引用队列，程序判断引用队列是否有虚引用来采取回收前的必要行动。

3. Java堆内存分配
   Java堆被划分成几个不同的区域，可以根据各个区域的特点选择合适的垃圾回收算法。
   jdk1.7中被分为新生代（eden区，两个survivor区from to），老生代，永久代（1.8中被元数据区取代，存放在本地内存中，永久代基本不被回收）
   1. eden区
      对象优先分配在eden区，当eden区没有足够空间进行分配时，虚拟机将发起一次MinorGC。 **每次minorGC之前都会检查老年代是否可以容纳所有新生代对象，即分配担保机制**（若老年代能容纳，则，将新生代对象提前转移到老年代，）
   2. 大对象直接存放在老年代，（即需要大量连续内存空间的对象），以减少新生代垃圾回收的频率。
   3. 长期存活的对象进入老年代。虚拟机给每个对象一个对象年龄计数器（在对象头中），**若对象在eden区进过一次MinorGC仍然存活且survior区可以容纳，则放入survivor区，并将对象年龄设置为1**，在survivor区每次存活年龄加1，**到15时被升入老年代。**且会进行**动态年龄判断，如果survivor区从小到大排序，当某个年龄对象总和大于survivor空间一半时，年龄不用到达15，在大于等于该年龄的对象都会提前进入老年代**
   4. GC区域：
      1. **MinorGC**：当新生代的eden区满了触发youngGC（**只收集新生代**，部分eden放入survivor区，youngGC后会有部分存活对象进入老年代），oldGC会对老年代回收（只有CMS有该模式），
      2. **fullGC**：**对整个java堆和方法区收集**。触发条件分配担保，**每次MinorGC之前会判断老年代剩余空间能否容纳所有新生代对象**，如果可以就进行MinorGC，如果不可以则判断是否允许分配担保失败（**分配担保**）。允许则检查老年代剩余空间是否大于历次晋升的平均大小，如果大于则尝试MinorGC（**把survivor无法容纳的eden区对象放到老年代中**，不考虑GC年龄）。如果小与则fulGC，如果不允许担保失败则fullGC。
   5. **什么情况会触发GC？**：
      1. eden区满了触发MinorGC，
      2. MinorGC之前进行分配担保（每次MinorGC之前会判断老年代是否能放下所有新生代对象，可以则MinorGC，不可以则看是否允许担保分配，允许则判断老年代空间和历次平均进入老年代对象的大小，大于则进行MinorGC，不允许或者担保失败了则进行fullGC）
      3. 永久代不足了也会触发fullGC
   
4. 对象死亡判断方法？  **引用计数法 可达性分析**
   **在GC前第一步要判断哪些对象已经死亡**
   
   1. 引用计数法：对象添加一个引用计数器，有一个地方引用则计数器加一，引用失效则计数器减一，计数器为零时对象就不能再被使用。但是会**发生循环引用的问题**，当两个对象相互引用，但是整体并没有被外部引用，此时这两个对象由于对象计数器不为0就不会被回收。
   2. 可达性分析算法：通过称为GC Root的对象最为起点，从这些节点根据引用向下搜索，称为引用链，当一个对象没有任何GC Root的引用链相连（即GC Root不可达），则表示该对象不可用，需要被回收。
      1. 哪些对象最为**GC Root**？java栈中局部变量表引用的对象，方法区中类静态属性引用的对象，方法区中类常量引用的对象，同步锁持有的对象，native方法引用的对象。
      2. **对象可以被回收，就一定被回收吗**？对象死亡会经历两次标记，可达性分析会对不可达对象进行一次标记并进行筛选，条件为是否重写finalize且没执行过，放入队列等待执行，执行后进行二次标记，如果在finalize方法中与引用链建立联系就移出回收集合，否则被回收。
      3. 字符串常量回收？:字符串常量池在1.7中存放在堆中，存放的是堆中字符串的引用，假如没有任何String对象引用该字符串常量，如果此时内存回收，该字符串引用被清理出常量池。
      4. 判断无用的类？：回收方法区中无用的类，同时满足三个条件可以被回收：堆中没有该对象的实例，加载该类的类加载器被回收，该类的class对象没有被引用，无法在任何地方通过反射加载该类方法。
   3. 二次标记
      第一次标记不在GCRoots引用链上的对象
      第二次会判断是否要执行finalize方法，进行小规模的标记，如果没有覆盖finalize方法或者已经调用过了则判断彻底死亡，如果有finalize方法与引用连上对象建立了引用，则不被回收。
   
5. 垃圾回收算法？

   1. 标记-清除算法
      标记所有存活的对象，标记完后将所有没标记的对象回收
      缺点：会存在碎片空间，效率不高

   2. 标记-整理算法
      标记所有存活的对象，标记完后将存活对象整理到一端，清理剩余区域
      缺点：效率太低，需要移动对象。适合老年代大量存活的场景

   3. 复制算法
      将内存空间分为两块，每次使用其中一块，当写满后，将存活的对象复制到另一块空间，再把使用的空间一次性清理掉。
      缺点：对空间的使用效率不高，当存活对象多时复制性能变差，不适合老年代

   4. 分代收集算法

      hotspot分为新生代、老年代、永久代每个代使用不一样的垃圾回收算法。eden区使用的就是复制算法（每次使用eden空间和其中一块survivor空间）。
      新生代每次都有大量对象死亡，存活对象少，使用复制算法。
      老年代每次回收有大量对象存活，使用标记整理或者标记清除算法。

6. 垃圾收集器
   垃圾收集器是对垃圾收集算法的具体实现，hotspot中实现了许多不同的垃圾回收器。
   jdk8中：新生代使用parallel scavenge，老年代使用parallel old
   jdk9中：默认使用G1
   
   1. serial收集器:**新生代**，单线程，需要暂停所有用户线程，使用**标记复制**算法。
   
   2. parNew：多线程并行版本的serial，**新生代**，会暂停所有的用户线程，使用**标记复制**算法。只有parNew能与CMS老年代配合工作（CMS以响应时间为目标）
   
   3. parallel Scavenge：多线程，**新生代**，会暂停所有的用户线程，使用**标记复制**算法，与parNew的区别是，**关注点为吞吐量（**高效利用CPU），可以自适应调节策略（自动调节新生代比例，年龄，内存大小等）。为jdk1.8默认新生代收集器
   
   4. serial old：**serial的老年代版本**，使用**标记整理算法**，单线程，暂停所有用户线程。作为cms发生failure（预留内存不够存放浮动垃圾）时后被收集器
   
   5. parallel old：**parallel scavenge的老年代版本**，多线程并发收集，与parallel scavenge配合使用，标记整理算法。**关注点为吞吐量**
   
   6. cms：concurrent mark sweep，**追求最短停顿时间**，**老年代**，**标记清除**，四个步骤。**初步标记**（找到与GC Root直接相连的对象，需要暂停用户线程，时间极短），**并发标记**（GC与用户线程并发，找到引用链上可达对象），**重新标记**（修正并发标记用户线程导致的标记产生变动的对象，需要暂停用户线程，时间比并发时间长，比并发标记段），**并发清除**（清除之前被标记为死亡的对象）。浮动垃圾问题使用serial old作为cms的备用（因为都是以暂停时间为目标）
   
   7. G1：哪块垃圾多就先回收哪块，**分代设计**（**但是不会为每个代分配固定大小的空间，以region为基本单位动态的分配，以及回收，每次回收都是region的整数倍空间**）并且使用记忆集来记录跨区域的引用关系，从而避免全堆扫描，每个region维护一个记忆集（其中记录其他region指向自己的指针）。**可以设定性能目标**（暂停时间）
      **并行回收时如何保证和用户线程互不干扰？**1、在回收过程中改变的引用关系处理，只按照快照中的引用来回收，2、回收过程中创建的新对象，G1为region中维护两个指针用于存储在回收过程中创建的对象。
      四个步骤：**初步标记**（**暂停线程**，标记GC Root直接关联的对象），**并发标记**（标记在引用链上的对象，生成快照），**最终标记**（标记在并发过程中快照引用的变动，**暂停线程，时间极短**），**筛选清除**（**暂停线程**，对回收价值和成本排序，根据用户设定的性能目标来制定回收计划，在将回收的region中存活对象复制到空region中再清除回收region）
   
      

#### 3 JMM（java内存模型）

**java多线程之间的通信由jmm来控制，jmm决定了一个线程对共享变量（即堆中的数据对每个线程是共享的）的修改何时对其他线程可见**，线程之间的共享变量存放在主内存中，每个线程都有一个私有的本地内存（对高速缓存、写缓存、寄存器的抽象），存放该线程读写共享变量的副本。然而在多线程并发时由于。**happens before控制线程间数据的可见性不控制线程执行顺序**

（编译器重排序（不改变单线程语义重排序语句）、指令及重排序（不存在数据依赖的并行处理）、内存重排序）可能会出现内存可见性问题，因此**JMM中定义happens-before（保证可见性与顺序性）规则与volatile（）、synchronized、final三个关键字**来控制重排序保证**可见性、有序性、原子性**。happens before可以保持一定程度的有序性和可见性，需要和这三个关键字结合使用。

- ==synchronized 关键字。==用于线程**同步**，指多个线程访问共享变量时按照一定的顺序进行操作，避免数据竞争与不一致问题 **1、同步方法，锁对象2、同步代码块，锁括号里指定的对象3、静态同步方法，锁当前类**。(**在同一时刻只有一个线程可以获得synchronized锁住的对象或类**)

- 本质：**加锁释放锁本质是获取一个对象的监视器**，这个过程是排他的（同一时刻只有一个线程可以获得synchonized锁着的对象的监视器），monitorenter获取成功则对对象进行访问，获取失败则进入同步队列，线程状态变成block，monit锁释放则同步队列中线程重新尝试获取监视器，

- synchronized是可重入的。同一个线程可以反复给同一个对象加锁，并不会因为上次加的锁还没释放而阻塞。获得锁监视器计数器加一，释放锁监视器计数器减一，计数器为0表示锁释放

- **对同一个监视器的解锁happens before对该监视器的加锁**。内存角度：解锁时将本地内存共享变量刷到共享内存，加锁时将本地内存设为无效，从共享内存中读取最新值。从而实现了数据的一致性
  **常常synchronized与wait结合使用，synchronized用来给对象加锁，wait用来让线程进入waiting状态实现等待通知，生产者-消费者模式**
  
  - 锁升级：偏向锁、轻量级锁（CAS）、重量级锁
  
    偏向锁：当某个共享资源第一次被线程访问时（无锁升级为偏向锁），将偏向锁标志为变为1，并将线程id加入对象头的锁信息中，当同一个线程再次访问时只需要比较偏向锁的线程id即可
    轻量级锁：当没有开启偏向锁或者在偏向锁延迟时间之前或者当前已经是偏向锁，但是第二个线程访问共享资源尝试获取偏向锁失败，升级为轻量锁，即通过CAS自旋来获得锁，
    重量级锁：当自旋超过一定次数还没获得锁，变成重量级锁，JVM会阻塞线程，等待执行完唤醒
  
    ![image-20240313160931468](C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240313160931468.png)
  
- volatile 关键字
  **保证所有线程看到被volatile修饰的变量的值都是一样的**，对**volatile变量的写操作happens before任意后续对这个变量的读操作**
  **写**vloatile变量时JMM会把该线程对应的本地内存中的共享变量立即刷写到主内存中，**读**volatile时JMM会把本地内存设置为无效，从主内存中读共享变量。**同时volatile会禁止部分重排序，保证执行顺序**

1. 如何减少上下文切换？：上下文切换指cpu通过时间片轮转执行任务，当一个线程任务执行一个时间片会保持当前线程状态并执行下一个线程任务，当切回该线程时会从保存的状态继续执行。多线程竞争锁，会造成线程阻塞，cpu资源分配给其他线程，会进行上下文切换。无锁并发编程减少线程间的竞争、CAS非阻塞不使用锁、自旋，**解决的核心是减少阻塞**。

2. 并发出现的问题以及原因？：上下文切换、死锁、内存泄露、线程安全。上下文切换带来的原子性问题，使用synchronized。缓存导致的可见性问题，使用volatile。重排序导致的顺序问题，使用happens before规则

3. 线程状态：
   [Java面试必会：线程如何在 6 种状态之间转换？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/130875882#:~:text=Blocked 与 Waiting 的区别是 Blocked 在等待其他线程释放 monitor,锁，而 Waiting 则是在等待某个条件，比如join 的线程执行完毕，或者是 notify ()%2FnotifyAll () 。)

   1. 新建状态NEW：线程实例被创建 new Thread后在start之前
   2. 运行状态Runnable：包括Ready、Running，当调用线程start方法、sleep方法结束，说明有运行的资格，可能在运行也可能在等待CPU资源，running即cpu调度处于ready状态的线程开始执行。即ready是进入running状态的唯一入口。
   3. 阻塞状态Blocked：当synchronized获取锁阻塞，没有得到monitor锁的时候，当获取到监视器的时候回到runnable状态
   4. 等待状态waiting:当Object.wait(),Thread.join(),Lock时线程会进入waitting状态，需要等待一个条件，如notify唤醒，或者join线程执行完毕，或unlock解锁。
      **waiting和timed_waiting线程要等待join线程执行完毕或unlock解锁才能进入runnable状态**
      **当其他线程调用了notify来唤醒，会进入blocked状态（因为还没获取到锁，要获取到锁才能runnable状态）**，
   5. 超时等待状态Timed_waiting:**sleep（）**超过时间后系统自动唤醒进入ready状态。当设置了时间的Thread.sleep(millis)、Object.wait(timeout)、Thread.join(millis)、Lock(nanos),即只要设置了超时时间的以上方法都使用Timed_waiting
   6. 终止状态TERMINATED:线程执行完毕(即run方法执行完毕)后或者出现异常导致线程终止
      blocked是在synchronized获取对象的monitor锁而阻塞的状态
      waiting是wait、join、lock中线程阻塞，需要等待其他线程执行特定的操作，即需要线程间的通信（如wait需要其他线程notify）

4. 创建线程 
   **实现Runnable和Callable接口的类当成一个在线程中运行的任务，还需要通过Thread来调用。**

   1. Runnable接口：重写run方法，将实现了run方法的实例传到Thread类中实现多线程 `Thread thread=new Thread（instance） thread.start（）`instance可以为实现了Runnable接口的匿名内部类
   2. Callable接口：可以有返回值，使用FurtureTask进行封装
   3. Thread类：继承Thread类，重写run方法，并start执行。

5. 线程中断

   1. interrupt用于中断调用的实例线程 `myThread.interrupt()`或 `Thread.currentThread().interrupt()`中断当前线程 ,设置线程的中断状态为true，**线程通过判断中断标志位执行决定如何中断，可以停止、延后或忽略中断，而不是立即中断从而保证数据的完整性**。 如果是sleep、wait等阻塞的线程，调用了interrupt中断，则会将中断标志位清除并且抛出interruptExecption异常，是由线程的try-catch方法抛出，用于唤醒线程并执行catch中的逻辑
   2. interrupted：静态方法，通过Thread调用，内部实现是调用当前线程的isInterrupted并且重置中断状态
   3. isInterrupted：实例方法，不会重置中断状态

6. 线程之间的通信

   - volatile 两个线程公用一个volatile修饰的变量，volatile 保证变量的可见性，一个线程对volatile 的修改，另一个线程可以立马看到
   - synchronized修饰的代码块或者方法在同一时间只有一个线程可以处于方法或代码块中，所以可以在同步块中访问共享变量，包装可见性和排他性。
   - wait/notify机制，如线程A和线程B协同工作，A工作完了通知B。**wait和notify在对象中，线程A调用对象的wait方法进入waiting等待状态，线程B调用对象的notify方法通知在对象中等待的线程，使得线程A从wait方法中返回，继续执行后续操作**  ==调用wait方法会释放对象的锁==
   - Thread.join(),线程A调用B.join(),表示A要等B执行完成才会接着执行，线程会处于waiting状态

   

7. Lock锁
   **Lock需要显示的lock与unlock创建锁与解锁，而synchronized不用显示的加锁。**

   **实现Lock接口的API基本都是通过AQS提供的能力来实现的，即API的功能基本都是调用AQS方法来包装实现的**，如ReentrantLock**对外提供Lock接口定义的Api，通过AQS来实现**，**Lock面相锁的使用者，定义了使用者与锁的交互接口，AQS面相锁的实现者，简化锁的实现方式**。Lock锁与Condition接口一起使用实现等待通知模式（类似于synchronized与wait结合），Lock继承AQS类，Condition接口提供类似于监视器方法，ConditionObject是AQS的内部类，由Lock对象创建出来 `Condition condition=lock.newCondition()`

8. AQS队列同步器
   是构建锁或其他同步组件的基础框架，仅仅定义了若干同步状态获取和释放的方法来供自定义同步组件（锁等）使用。子类通过继承AQS并实现他的抽象方法来管理同步状态，**子类被推荐作为自定义同步组件的静态内部类**。
   **AQS使用volatile的int变量来表示同步状态，使用队列来进行线程的排队工作，核心思想是，如果被请求的资源空闲，那么将当前请求的线程设置为有效的工作线程，将共享资源变为锁定状态，如果共享资源被占用，那么用阻塞等待唤醒机制来保证锁的分配，将暂时获取不到锁的线程加入到队列中**（队列中每个节点包括waitStatus、prev、next、thread）

   - 加锁流程：
     - 调用ReentrantLock的lock方法，
     - lock方法会调用sync内部类的lock方法（sync内部类是继承了AQS的公平锁或非公平锁）
     - lock方法先进行cas设置state为1（获取锁），成功则将当前线程设置为工作线程
     - lock方法cas失败则调用AQS的acquire方法
     - acquire方法会执行tryAcquire方法，会执行ReentrantLock中内部类实现的tryAcquire方法
     - tryAcquire调用notfairTryAcquire再次进行cas抢锁或重入锁或者执行AQS后续的排队操作。

9. 并发安全容器与框架

   1. ConcurrentHashMap 在concurrent包中

      为什么用？：HashMap是线程不安全的，HashTable是线程安全的，但是使用了synchronized，锁住整个对象（get方法加锁），即会锁表，使得效率低下。
      concurrentHashMap使用锁分段技术，每把锁用于锁容器的一部分数据，当多线程访问容器里不同数据段的数据时，线程之间就不存在锁竞争了。

      - jdk1.7 
        由segment数组结构和hashentry组成，segment将Hash表划分为多个分段，在执行put操作时根据hash算法定位到元素所在的segment并加锁，是可重入锁（继承ReentrantLock），扮演锁的结构，每个segment内部可以看成一个hashmap，hashentry中存储键值对数据
        **get操作**：通过hash算法得到对应的segment，再根据hash值得到对于的hashentry数组结构，并对hashentry对象进行链表遍历找到对应对象，**get方法不需要加锁（并没有显示的加锁），hashentry中的共享变量都设置为volatile，保证可见性，每次获得的都是最新值**
        **put操作**：先定位到对应的segment，对segment获取锁（获取失败先进行自旋，即不阻塞线程，超过阈值阻塞获取），以保证线程安全。第一步先判断segment中的hashentry是否需要扩容，第二部定位put的位置，并更新。
      - djk1.8
        使用**数组+链表+红黑树实现**，加锁通过CAS和synchronized。相同Hash值的元素链表长度大于一个阈值且满足一定的容量要求时，把链表转换为红黑树，进一步提高查找性能。
        **put操作：**先计算hash值，判断Node数组是否需要初始化，定位到Node，获取首节点f，f为空则自旋添加，f在扩容则参与扩容，若都不是则synchronized锁住f节点，通过红黑树或链表遍历插入
        **get操作**；通过hash值找到对应的首节点，如果该数组为空，返回null，若该节点就是需要的值，返回该节点，若为红黑树或正在扩容使用find，若为链表则遍历查询。

   2. CopyOnWriteArrayList 并发的List
      修改时先copy容器，复制出新的容器再修改新容器，修改完成后将引用指向新的容器。适用于**读多些少**的场景或**读性能很高**的地方。在**迭代期间允许修改**（因为迭代使用旧数组）

   3. ConcurrentLinkedQueue 使用cas非阻塞的线程安全队列
      定位到队列的尾部，使用cas将该节点设置为尾节点的next节点，不成功则充实，适用于并发不是特别剧烈的场景

   4. BlockingQueue 阻塞队列
      插入方法：若队列满了阻塞到不满为止
      移除方法：若队列空了阻塞到不空为止
      若队列为空，消费者会一直等待，当生产者添加元素时，消费者如何知道当前队列有元素呢？使用等待通知模式，使用了Condition来实现

   5. fork/join
      并行执行框架。把大任务分割成若干个小任务，最终汇总小任务的结果得到大任务的结果。
      工作窃取：大任务分割成若干个户不依赖的独立子任务，子任务放到若干个队列中执行，每个队列对应一个线程，当本线程队列的任务执行结束，从其他线程的队列中获取任务执行，队列为双端队列，窃取从尾部窃取。要继承Recursive重写compute。使用fork方法会将子任务放入compute中，查看子任务是否需要再分割或执行，执行完返回结果。使用join阻塞获得结果

10. java原子类和并发工具类

   11. cas：compare and swap 两个参数预期旧值和新值，cas为乐观锁，**假设在操作期间旧值没有发生改变**，若没有变化则更新新值，否则不更新新值。

       但是cas会发生**ABA的问题**（即旧值从A变成B又变成A），以及**只能保证一个共享变量的原子操作，而不能对一系列操作保证原子性**（如果多个操作之间会存在被其他线程打断的可能），使用版本号，即每次更新操作都把版本号加1。

   12. 原子操作类 Atomic包

       1. 原子更新基本类型类 AtomicInteger AtomicBoolean AtomicLong 原子的方式更新基本类型
       2. 原子更新数组 AtomicIntegerArray AtomicLongArray AtomicReferenceArray 原子的方式更新数组中元素
       3. 原子更新引用类型 
       4. 原子更新字段类型 AtomicStampedReference(原子的更新带有版本号的引用类型) 原子地更新某个类里的某个字段

   13. 并发工具类

       1. CountDownLatch**可以使得一个或多个线程等待其他线程完成后再执行** 允许一个线程或多个线程等待其他线程完成操作。其中定义了一个计数器和阻塞队列，构造函数接受int参数标识等待n个线程完成，调用countDown方法时n会减1。countDownLatch.await方法会阻塞当前线程直到n变成0才唤醒。
       2. 同步屏障CyclicBarrier  让一组线程达到一个屏障时被阻塞，知道所有线程达到屏障时，所有线程才会继续执行。await方法会阻塞线程直到所有线程达到屏障
       3. semaphore控制并发线程数 控制同一时刻访问特定资源的并行线程数量，acquire获取许可证（若没有获得到则阻塞），release释放许可
       4. Exchanger线程间交换数据 提供一个同步点，两个线程在同步点可以交换数据，两个线程都执行exchange后可以交换数据，exchange的参数为要传递给对方的数据，返回值为接收到的数据

14. java线程池和Executor框架

    1. 为什么用线程池？
       降低创建、销毁线程的资源消耗，重复利用已创建的线程
       提高响应速度

       一般用于执行多个不相关联的耗时任务

    2. Executor线程池框架三个组成部分

       - 任务：主线程创建实现了Runnable和Callable接口的任务对象（实现了这两个接口的任务才能被线程Thread 、ThreadPoolExecutor 、ScheduledThreadPoolExecutor执行 ）
       - 任务的执行 即ThreadPoolExecutor 、ScheduledThreadPoolExecutor（继承了ThreadPoolExecutor并实现ScheduledExecutorService接口）
       - 异步计算结果Future（FutureTask类实现Future接口 **任务提交给Executor执行时返回一个FutureTask对象**  **submit方法执行会返回future对象，execut不会返回对象** 
       - 主线程调用Future对象的get方法获取执行结果，或调用cancel取消执行
       - <img src="https://javaguide.cn/assets/Executor%E6%A1%86%E6%9E%B6%E7%9A%84%E4%BD%BF%E7%94%A8%E7%A4%BA%E6%84%8F%E5%9B%BE-PBioDAvY.png" alt="Executor 框架的使用示意图" style="zoom:50%;" />

    3. ThreadPoolExecutor

       - 构造方法参数：corePoolsize核心线程数量，maximum最大线程数量，keepAliveTime空闲线程最大存活时间，任务队列BlockingQueue存储等待执行的任务，threadFactory线程工厂，handler拒绝策略
       - 拒绝策略：当已经达到最大线程数量且任务队列已经满了时进行拒绝。AbortPolicy抛出异常来拒绝任务，CallerRunsPolicy，由调用execut方法的线程来执行被线程池拒绝的方法，DiscardPolicy不处理任务，直接丢弃，DiscardOldestPolicy抛弃最早未处理的任务请求。默认为AbortPolicy。
       - 线程池创建：1 直接通过ThreadPoolExecutor构造函数创建自己定义参数、2 使用Executors工具类来创建多种类型的ThreadPoolExecutor（如固定线程的、单一线程的、CachedThreadPool、定时线程等），但是不推荐用工具类创建，可能会导致线程池规则设置不清楚资源耗尽（如固定线程的线程池使用无界的队列作为任务队列，可能会堆积大量的请求导致，CachedThreadPool最大线程池数量为Integer.MAX_VALUE，可能导致创建大量线程）。
       - 线程池常用的阻塞队列：`LinkedBlockingQueue`（无界队列）用于Single和Fixed。`SynchronousQueue`（同步队列）用于CachedThreadPool无限创建线程。
       - 原理分析：for循环生成10个任务worker，MyThreadPoolExecutor.execut(worker)交给核心线程为5，最大线程为10，任务队列ArrayBlockingQueue容量为100的线程池MyThreadPoolExecutor去执行,因此线程池先创建5个核心线程同时执行，剩下的5个加入等待队列，当核心线程有任务被执行完，线程就会拿新的任务执行。
       - execute（worker）流程：
         - 如果当前运行线程小于核心线程数，则创建线程执行，
         - 如果当前运行线程数大于了核心线程数，则加入同步队列中等待执行，
         - 如果加入失败（队列已经满了），而当前线程数小于最大线程则创建线程进行执行
         - 否则执行拒绝策略（即任务队列已经满了而且线程数已经达到最大）
           ![图解线程池实现原理](https://oss.javaguide.cn/github/javaguide/java/concurrent/thread-pool-principle.png)

    4. 对比

       - execut和submit  
         execut提交不需要返回值的runnable任务。subimt提交需要返回值的callable任务，线程迟返回一个Future类型对象，**调用Future的get方法会阻塞当前线程直到任务完成获得结果**，或get（long timeout，TimeUnit unit）指定等待结果的时间，超过时间还没得到结果则报错。

15. ThreadLocal 线程隔离，
    **线程本地变量是每个线程私有的，当线程访问ThreadLocal变量时会在每一个线程创建单独的变量副本，避免因为多线程操作共享变量导致数据不一致(因为每个线程操作的都是自己拷贝的ThreadLocal变量)，ThreadLocal可以在线程的任何地方被访问**
      以ThreadLocal对象为key，任意对象为值，在每一个Thread中维护ThreadLocalMap，当一个k-v被存储后，会一直附带在线程上，所以可以在线程的任意位置通过这个ThreadLocal对象取到存入的值。ThreadLocal对象.set(val)设置值，ThreadLocal对象.get()获取值。
      使用：如在一个线程中创建ThreadLocal变量，一个方法设置值为当前时间，另一个方法用当前时间减去ThreadLocal变量的值，得到方法的运行时间

    - 原理：**每个人Thread中有一个ThreadLocalMap的实例变量，其中每个entry的key是ThreadLocal对象的弱引用，v为存入的对象**，即每个线程都会向自己的ThreadLocalMap中存，实现线程隔离。ThreadLocalMap以数组实现。 

    - 使用时将thread local变量设置为private static将状态与线程关联起来，static保证只有一个threadloacal对象实例，private让threadlocal只能在类内部被访问，
    - 场景：**保存线程的上下文信息**(如某线程从controller层运行到了service层，service层想使用controller层的request，可以将这个request放入threadloacal中。对象跨层传递，而不用显示的通过参数传递)，**记录执行时间**，**数据库连接池用threadlocal保存数据库连接对象**，每个线程从连接池获得连接时，将连接对象存在ThreadLocal中，即该线程无论调用哪个方法，都用同一个连接，实现跨方法的事物控制。
    - 内存泄露风险：**ThreadLocalMap中Entry的键是弱引用**（每次GC无论是否内存充足若该对象只有弱引用的话会被回收）**当只有Entry中的key指向ThreadLocal时，ThreadLocal会被回收，此时Entry的key为null，但value不为null，因此有内存泄露的风险，因此在threadLocal使用完之后应当调用remove方法进行清理**而且threadlocal的get、set方法都会清除key为null的entry的value。
      即key为弱引用没有对象引用threadlocal时key被回收，key为null的value会在get、set方法时被回收
    - 为什么使用弱引用：因为如果使用强引用，如果调用threadlocal的对象被回收了，但是threadlocalmap中还有threadlocal的强引用，那么就一只不会被回收，直到线程结束。
    - <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240331180352356.png" alt="image-20240331180352356" style="zoom:50%;" />

 

   GC后key是否为null：因为使用的是弱引用，会被回收

13. 线程优先级
      操作系统采用时间分片的形式来分配处理器资源给线程，一个线程用完了一个时间片就会发生线程的调度，即便还没执行完，要等待下次分配。因此一个线程分到的时间片越多就能使用更多的处理器资源，线程的优先级就来控制线程所能获得时间片多少的能力。使用setPriority来设置优先级，最高为5。
      场景：经常遇到阻塞的线程（如IO操作比较多）可以设置高优先级，**而计算较多，消耗cpu的线程设置为低优先级，避免占用处理器**

#### 4 基础

1. Exception和error区别？
   [02 Exception和Error有什么区别？-极客时间 (lianglianglee.com)](https://learn.lianglianglee.com/专栏/Java 核心技术面试精讲/02  Exception和Error有什么区别？-极客时间.md)
   二者都继承了Throwable类，只有Throwable类的实例才可以抛出（throw）或捕获（catch），二者体现了对异常的分类，**Exception是可以预料的异常（应当被捕获）**，而**error是指正常情况下一般不会出现的异常**（如内存溢出、栈溢出等），绝大部分会导致程序处于不可恢复的状态（**程序无法处理**），Exception又分为**可检查**和**不可检查**异常（**即编译时异常和运行时异常**）。可检查的异常需要在源代码中显示的catch捕获或者抛出（IO相关、ClassNotFound）。运行时异常**RuntimeException通常为逻辑问题**，在运行时抛出，如数组越界，空指针异常等，**是可以通过修改逻辑来避免的，因此可以选择性的抛出**。

   [java之异常（Exception）与错误（Error）的区别_java中error和exception的区别-CSDN博客](https://blog.csdn.net/single_0910/article/details/114258581)**throw是主动抛出异常，throws是声明本方法异常（若产生异常将异常向上抛），finally一定会执行，且在return之前执行**

2. 谈谈final、finally、 finalize有什么不同？
   final可以用来**修饰变量、类、或者方法**。当修饰方法时，表示该方法无法被重写。修饰变量时如果该变量为基础类型则无法修改值，如果是引用类型则引用变量指向的地址不能修改。修饰类时表示该类无法被继承（如string类，保证平台的安全性，防止APi被修改）（==可以延伸到string类不可变的原因，再延伸到字符串常量池==）
   finally用于try-catch中，其中包含的代码块无论是否报错必须被执行。**如释放锁、关闭数据库连接等**
   finalize用于GC垃圾回收时保证对象被回收前完成特定的资源回收。是Object类中的一个方法，现在已经不推荐使用。

3. 强引用、软引用、弱引用、虚引用有什么区别？

   用于对象的生命周期以及GC垃圾回收中。
   强引用：我们一般new对象或者普通的对象引用都是强引用，只要对象还存在强引用就不会被回收。
   软引用：软引用需要显示的使用创造softReference。当内存不足是才会回收。在内存溢出之前回收。用于保存敏感的缓存。

   弱引用：weakReference只要有GC就会被回收。用于维护没有特定约束的关系。维护非强制的映射，如果对象还在就使用，对象不存在就重新实例化。

   虚引用：不能用于访问对象，**只用与引用队列联合使用来标记对象是否还存在**
   **引用队列：ReferenceQueue<T>** 

   `WeakReference<SavePoint> savePointWRefernce = new WeakReference<SavePoint>(savePoint, savepointQ);`savePoint为一个强引用指向的对象，savepointQ为该类的引用队列，**创建非强引用时可以指定一个引用队列，在非强引用（虚引用）指向的对象被回收后，会把该引用变量放入引用队列**，**为了提醒程序员某个类的非强引用型变量所引用的那个对象已经具有不可达性了**，再也不能获得堆中的那个对象了

4. String、StringBuffer、StringBuilder有什么区别？
   String是不可变对象（其使用了final修饰了类和属性，且属性是private，且没有对外提供setter接口）每次修改都会生成新的对象。
   StringBuffer时线程安全的（在方法上加synchronized关键字实现），且可以使用append和add对字符串进行修改或拼接
   StringBuilder是线程不安全的，但性能更高
   intern函数，将字符串放入字符串常量池中，如果字符串常量池已经有该字符串的引用，那么就返回引用，如果不存在就在字符串常量池中创建该字符串的引用并且返回

5. **反射与动态代理？**

   反射是指在java程序运行的过程中可以操作类或者方法（获取类属性与方法，对象的方法或构造对象等），**可以灵活的操作在运行时才可以确定的信息**。使用**反射前要获取到类的class对象**（类.class,对象.getClass,Class.forName(全类名），**比如在泛型中参数传入一个泛型Class<T>类并在类中创建该泛型类的对象,但是只有在运行时才知道具体类型，因此可以使用反射获取该Class类的对象**）

   **动态代理可以在不修改原有代码的情况下，在方法调用前后插入自定义的逻辑**。不需要知道类名就可以创建对象，使用反射包reflect下的**Proxy类和InvocationHandler接口**。首先创建接口（**代理类和被代理类都要实现接口，接口中定义要代理的方法**）创建类实现InvocationHandler接口，重写其中的invoke方法（即写代理对象调用方法前后的逻辑。调用Proxy类的newProxyInstance来创建代理对象（**参数为指定类加载器loader加载代理类，指定接口Interface即有什么方法，实现了代理逻辑的InvocationHandler类**），调用代理对象的方法。

   ==即通过Proxy类的newProxyInstance生成的代理对象调用方法时，实际上调用的是实现了InvocationHandler接口的类的invoke方法==

   为什么反射会比new实例化慢很多？： `new B(); 和 Class<B> c=B.class; c.newInstance()`因为 1.编译器无法进行优化 2.**被调用的class需要进行查找（按照类名或方法进行匹配）**3.检查是否存在无餐构造函数，调用构造函数代码，分配空间。而new实例化只需要分配空间和调用构造代码。

   **动态代理（分为jdk代理和CGLIB第三方依赖）（要实现==InvocationHandler接口重写invoke方法，与Proxy类newProxyInstance生成代理对象==）可以使用反射类Method来指定调用的方法**，**在不修改原对象的前提下，扩展对象的功能**，InvocationHandler（proxy，method，args）newProxyInstance（类加载器，接口，代理逻辑）。**jdk代理只能代理实现了接口的类，而CGLIB没有限制**，动态代理比静态代理更灵活（不用实现接口），不用针对每个被代理类写一个代理类。静态代理在编译时就将接口、实现类、代理类变成class文件，而动态代理在运行时动态生成类的字节码并加载到JVM中。**InvocationHandler和newProxyInstance都使用到了反射**

   静态代理？：1、定义要代理的方法接口2、被代理类实现接口3、被代理对象注入代理类，并且实现接口4、调用代理对象中的接口方法。 缺点：每个方法增强都是手动完成的，一旦接口发生变化，代理类也要进行修改，每个被代理的类都要实现一个代理类

6. int和Integer有什么区别？
   int是基础数据类型，Integer是int的包装类，其中包括了一些操作方法（如类型转换）。默认缓存了-128-127之间的数据。自动装箱为编译器自动调用Integer.valueOf将数值转化为Integer对象。

7. 对比Hashtable、HashMap、TreeMap有什么不同？ ****
   都是对Map接口的实现，HashTable线程安全，使用表锁。HashMap并不是线程安全的，TreeMap基于红黑树实现（有序）

8. **hashmap的扩容机制**
   数组与链表的形式存储，数组被封为一个个桶，通过哈希值决定在数组上的位置，相同哈希值的键值对被存放在同一个桶的链表上，**链表超过阈值会变成树**

9. try with resource代替try catch finally？
   try with resource中**catch和finally在try中资源关闭后运行**，在要关闭资源时（如IO流，scanner）优先使用try with resource

10. 泛型

    [Java 基础 - 泛型机制详解 | Java 全栈知识体系 (pdai.tech)](https://pdai.tech/md/java/basic/java-basic-x-generic.html#泛型类)
    **通过泛型参数可以指定传入对象的类型，编译器对泛型参数进行检查**提高代码的可读性，同时使用泛型后，编译器会对返回类型进行自动转换（如ArrayList<Person>，返回对象会从Object转为Person类型）
    泛型类、泛型方法、泛型接口。

    - 泛型类在定义时 `public class Generic<T>`**在类名后加加上<T>表示为泛型类**表示该类中使用泛型 **（变量的类型为T、方法的返回类型为T、方法参数为T）**如`private T key`，实例化泛型类时要指定具体的类型 `如Generic<Integer> gen=new Generic<Integer>();`
    - 泛型接口在定义时和泛型类一样 `public interface Generic<T>`表示该接口中有（**方法返回类型为泛型**、**方法参数为该类型** ）`如public T method(); 或者public void method1(T t);`，**在实例化泛型接口的时候可以指定类型也可以不指定类型（即为泛型类）**
    - 泛型方法 `public static <E> void printArray(E[] array)`，**声明方法时泛型<T>加在返回类型前表示为泛型方法**，可以使泛型方法（参数为指定类型、方法中局部变量为指定类型、返回对象为指定类型），调用方法时 `printArray(intarray)`或 `printArray(doubleArray)`
    - **项目中哪里使用泛型？：在返回数据给请求时，返回的实体类为一个泛型类，可以动态指定返回给请求实体类中的数据类型**

11. api和spi
    程序的不同模块之间通过接口来进行交互，实现了解耦，api由实现方提供接口规则，spi由调用方提供接口规则。spi由调用方提供接口给实现方去实现，在修改实现的时候并不需要去修改调用方。
    如slf4j是java的日志门面（接口），其实现可以是log4j或者logback等，换不同的实现是不用更改程序代码。
    **spi机制：**在类加载的时候会先去找到与class类相对应的在src目录下`META-INF/services`这个文件下相同全类名的文件并加载到内存中，然后根据相对应的这个文件中的内容找到spi接口的实现类（同名文件中的内容为实现类的全类名），之后就可以**通过反射机制去生成对应的实现类对象**（使用ServiceLoader加载spi的实现类 ServiceLoader<接口名> loader=ServiceLoader.load(接口名.class)，loader为一个实现类的集合，因为文件的内容中可能有多个实现类）
    问题：会遍历加载所有实现类，不能按需加载。多个ServiceLoader同时load时会有并发问题

12. 序列化和反序列化
    **当需要将java对象保存在文件中或者在网络中传输java对象，需要使用序列化技术**
    序列化：将数据结构或对象转换成二进制的字节流。JSON和XML为文本的序列化方式，可读性交好但性能较差
    场景：远程方法调用RPC将对象网络传输，将对象存储到文件之前，对象存到数据库
    在TCP/IP模型的表示层（应用层）中，该层进行数据处理（解码编码，加密，压缩解压缩）
    当不想被序列化的遍历用transient修饰（只能修饰变量，不能修饰类或方法），static修饰的变量属于类，不属于对象，所以也不会被序列化
    jdk自带的序列化方式serializable，效率比较低而且存在安全性问题。（private static long serialVersionUID是序列化版本，与当前类进行比较是否一致）

13. 值传递和引用传递 **Java中只有值传递** 
    [一文彻底搞懂引用传递与值传递~ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/450013637)

    将实参传递给函数的时候，
    值传递：**函数接收到的是实参的拷贝值**（==形参是实参的拷贝==），引用遍历时形参收到的是实参引用地址的拷贝值，
    引用传递：传引用，**函数收到的是实参引用的对象在堆中的地址的**（==形参就是实参==），不会创建副本，**对形参的修改会影响到实参**。
    **参数为基本类型时**：传递的是值的副本，`int a=1; int b=2 swap(a1,b1)`，a和b的值并不会变化，swap只收到了复制来的值（将对实参的拷贝值进行了交换，即a1和a是指向同一个地址的两个变量）
    值传递参数为引用类型时：**值传递的是引用地址的副本，即形参和实参指向同一块地址的两个变量**（**一个是实参的地址，一个是对象在堆中的地址**）因此对形参的修改会影响到实参。 `String A="sss"; String B="ccc" swap(a,b)`,A和B为引用类型，swap形参a和b收到A和B的地址，只是交换了形参a和b指向的地址，而实参A和B指向的地址并没有变<img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240311160610759.png" alt="image-20240311160610759" style="zoom:50%;" />

14. 浅拷贝和深拷贝

15. 语法糖
    方便开发的特殊语法，如for-each、自动装箱拆箱、泛型、try-with-resource、lambda表达式，在被JVM执行前需要被编译器解糖，即转换为JVM认识的语法

16. 向上转型，向下转型

    `Animal animal = new Dog();`    // 向上转型，父类应用指向子类对象,向上转型不比强制转换

    `Dog dog = (Dog) animal;` //向下转型，子类引用指向父类引用，**向下转型必须要强转**
    **如果父类应用本身指向的是子类对象，则向下转型是安全的，但是如果父类引用指向的是父类对象，编译会通过，但是运行的时候会报错**，最好使用instanceOf先判断引用指向的是否是子类对象
    `if(animal instanceOf Dog){...}`
    ==**静态方法调用的是引用变量的，因为静态方法在类加载时就已经绑定**==
    ==**成员变量调用的是引用变量的**==
    ==**实例方法调用的是实际对象的，因为是动态绑定的**==

17. 类加载
    **虚拟机将class文件中描述类的信息加载到内存中，变为可以直接使用的class对象**

    - 过程包括：加载、验证、准备、解析、初始化
    - 加载：通过类的全限定名获取类的class文件二进制字节流，将二进制流转化为方法区中运行时数据结构，在堆中生成代表该类的class对象（**作为访问方法区中数据的入口，即可以通过class对象访问这些数据**）（**类加载器不需要等到某个类被首次使用才加载，若加载时遇到了问题，则会在首次使用类时才报出错误**）
      通过类加载器完成加载步骤（可以通过自定义的类加载器完成）
    - 验证：确保加载的class文件字节流符合虚拟机的要求（**但验证这一步不是必须执行的，确保安全的情况下可以关闭以加快类加载的过程**），包括文件格式验证、符号引用验证、语意是否合法、元数据验证
    - 准备：为类的静态变量分配内存，并且初始化为默认值（分配在方法区中）
    - 解析：把类中的符号引用转化为直接引用（将字面量转化为在内存中的位置），**解析阶段的发生顺序由于动态绑定存在可能是不确定的，有可能会发生在初始化之后，因为可能加载类的时候该引用为一个接口（多态），实际上引用的对象是不确定的，因此只能等到运行时才能知道具体的类才可以将符号引用变为直接引用**
    - 初始化：为类的静态变量赋予正确的初始值（通过静态代码块或者声明时的指定值）（**只有当对类主动使用时才会类的初始化**）
    - 类的卸载：即class对象被GC。满足三个条件：该类所有实例对象被GC，该类没有在任何地方被引用，该类的类加载器被GC。**因此jvm自带类加载器加载的类不会被卸载。**

18. 类加载器
    类加载器负责加载类的字节码（.class文件）到JVM中（生成一个class对象），每个类都有一个引用指向加载它的ClassLoader。**赋予了Java类可以被动态的加载到JVM中并执行的能力。**
    JVM启动时不会一次性加载所有类，会在使用时或者JVM预测会使用时进行加载

    **除了BootStrapClassLoader，其他类加载器都继承自ClassLoader，每个类加载器可以通过getParent获取其父加载器（不是继承关系），若获取到的classLoader为null则说明该类是通过BootStrapClassLoader加载的，类加载会自底向上查找类是否加载，会自顶向下尝试加载类**

    - BootstrapClassLoader（启动类加载器） 加载JDK的环境变量%JAVA_HOME%目录下的核心类库（如util、io、lang、math包），无法被java程序直接应用，由c++实现
    - ExtensionClassLoader（扩展类加载器）加载扩展库，加载（环境变量%JRE_HOME%目录下的）jre/lib/ext下的扩展包，开发者可以使用扩展类加载器
    - ApplicationClassLoader （应用程序类加载器），**加载当前应用classPath下的所有jar包和类。**  A.class.getClassLoader(class类的getCLassLoader方法获取类加载器)
    - 自定义类加载器 ，需要继承java.lang.ClassLoader抽象类。 `Class clazz=Class.forName(Name,true,ClassLoader)或者loader.loadClass(name，true)`**来使用自定义类加载器,返回值为class对象**，loader.loadClass只加载类，不执行静态代码块（初始化）。Class.forName会执行全部类加载流程
    - 为什么要自定义类加载器？**隔离加载类，修改类的加载方式，防止源码泄露（对.class文件加密，用自定义类加载器解密）**，隔离类是指，若两个类类名相同，但加载器不同，则java判定这两个类不相同。
      继承ClassLoader,关键方法为loadClass()（实现了双亲委派机制，其中会调用父类的loadClass和本加载器的findClass方法），findClass（建议重写该方法，无法被父类加载器加载的类会通过这个方法加载）
    - 双亲委派
      在classLoader的loadClass方法中，
      1 **先判断类是否已经被加载过**，已经加载过会直接返回。
      2 类加载器调用父类加载器的loadClass方法进行加载类，请求会一直到达启动类加载器BootstrapClassLoader，
      3 当父类加载器无法加载该类时，当前类加载器才会尝试使用自己的findClass来加载类。
      好处：1、**避免类被重复加载，也保证了Java核心API不会被篡改，如自己写的Object类要被加载，但是启动类已经加载过Object了，就不会加载自己写的Object类**
    - 如何打破双亲委派：在loadClass（）方法中会先判断该类是否已被加载，再交给父类的loadClass方法，若父类无法加载则才调用本类的findClass方法，因此要打破双亲委派，需要重写本类的loadClass方法（可以先调用本类的findClass方法）。
    
19. 对象加载过程


20. this和super

   - this()只能用于构造方法的第一行，表示当前构造方法复用对应的构造方法，代码复用。
   - super()只能用于构造方法的第一行，表示当前构造方法调用父类对应的构造方法

21. IO流
    数据输入到计算机内存的过程即输入，反之输出到外部存储（比如数据库，文件，远程主机）的过程即输出。所有IO类的都继承InputStream/OutputStream/Reader/Writer抽象基类，Stream为字节流，Reader为字符流

    1. byte和char有什么区别？

       - byte为字节（-128,127），char为字符（0-255）（unicode编码中占用两个字节），

       - char可以表示中文，byte不可以
       - 对于英文字符，int byte char可以相互转换（英文的ASCII码在0-127之间）

    2. InputStream/OutputStream字节流 操作音频、图片等媒体文件

       - `FileOutputStream output = new FileOutputStream("output.txt")` 有write、close方法
         `InputStream fis = new FileInputStream("input.txt")`有read、available、skip、close方法
       - 二者通常会配合`BufferedInputStream`和`BufferedOutputStream`一起使用
         `BufferedInputStream bufferedInputStream =newBufferedInputStream(newFileInputStream("input.txt"));`
       - `DataOutputStream` 用于输出指定类型数据，不能单独使用 `dataOutputStream.writeBoolean(true);`
         `DataInputStream` 用于读取指定类型数据，不能单独使用 `dataInputStream.readBoolean();`

    3. Read/Write字符流 可以直接读字符内容，使用字节流读字符文件时很容易出现乱码，字节流再转字符比较耗时。

       - 默认采用unicode编码（中英文字符占2个字节），utf8编码（英文1字节，中文3字节）
       - FileReader可以直接操作字符文件 `FileReader fileReader = new FileReader("input.txt");`
          FileWriter 可以直接将字符写入到文件 `Writer output = new FileWriter("output.txt")`

    4. 字节缓冲流  BufferedInputSream、BufferedOutputStream与InputStream/OutputStream配合使用
       **IO操作很消耗性能，使用缓冲流将数据加载至缓冲区，一次性读/写多个字节，避免频繁IO操作**，使用装饰器模式来增强InputStream/OutputStream子类对象的功能。字节缓冲流在调用write()和read()这两个一次只读写一个字节的方法的时候，会先将读写的字符放在缓冲区，大幅减少IO次数，提高读写效率。

       - BufferedInputSream
         不会一个一个读取字节，而是先将读取到的字节缓冲起来，再从缓冲区中一个个的读字节
       - BufferedOutputStream
         不会一个一个写入字节，而是先将字节写到缓冲区中，

    5. 字符缓冲流

       - `BufferedReader` （字符缓冲输入流）和 `BufferedWriter`（字符缓冲输出流）类似于 `BufferedInputStream`（字节缓冲输入流）和`BufferedOutputStream`（字节缓冲输入流）

    6. 随机访问流 RandomAccessFile
       使用seek(long pos)来指定偏移量，使用getFilePointer得到当前位置。**多用于大文件的断点续传**（将文件切为多个小分片，当上传失败，只需要重新上传失败的分片，不用全部重新上传）

    7. 适配器模式 （适配器继承目标类并且封装适配者对象）
       适配器模式，分为类适配器和对象适配器，类适配器使用继承关系来实现，对象适配器使用组合关系来实现，**使得接口互不相容的类相互合作**（比如一个类A方法只处理字节类B，现在只有字符类C，那么类A和类C就不兼容，使用适配器Adapter将字符类C编码为字节类B，则类A就兼容适配器Adatper编码的对象C了），**适配器继承类B，并且类中封装一个C对象，将类B转化为类C**。IO流中字符流和字节流接口不同，通过适配器可以将字节流对象适配成一个字符流对象，这样可以直接通过字节流对象来读写字符数据，就使用了对象那个适配器。

    8. 装饰器模式  (扩展某一类所有子类的功能)
       **装饰器通过组合对象来替代继承实现扩展原始类的功能**，==装饰器类需要和原始类继承相同抽象类或实现相同接口，并且内部持有一个被装饰对象的引用==（**装饰器类的构造函数一个参数为需要被装饰类的对象，这样被装饰类的所有子类都可作为参数被这个装饰器扩展功能**，），同时又比生成子类更为灵活（一般的思路为对类A扩展一个功能的话，将功能写为一个接口C，使用子类B继承类A的每个子类并实现接口C，对类A的每个子类都要写一个单独的类，因为类A每个子类方法的实现都不同）

22. IO模型 

    [详解磁盘IO、网络IO、零拷贝IO、BIO、NIO、AIO、IO多路复用(select、poll、epoll)_nio、aio、零拷贝、io 多路复用机制-CSDN博客](https://blog.csdn.net/qq_37555071/article/details/113932533)
    （磁盘IO和网络IO，网络IO监听网络连接事件 可读 可写 错误等，磁盘IO监听文件描述符状态 可读 可写等，都是监听IO操作的状态来提高系统的并发性能）
    为保证系统的安全性和稳定性，一个进程的地址空间分为用户空间和内核空间，一般的程序运行在用户空间，文件管理、进程通信、内存管理这些系统级别的资源操作都是运行在内核空间，因此程序要进行IO操作要系统调用，由系统去具体执行IO操作。**IO分为两个阶段，第一个阶段判断是否有事件发生，第二阶段是有事件发生后，执行真正的读写操作，将数据从内核空间拷贝到用户空间**

    1. 读操作：收到读的Io，操作系统先检查内核空间是否存在需要的数据，如果内核空间已经有了那么直接把内核空间中的数据copy到用户空间返回给程序，如果内核空间没有，对于磁盘IO，会从磁盘中读到用户空间，对于网络IO的读操作（指程序从磁盘或者网卡（socket）中读），read进行系统调用，从socket中读时，程序要等待客户端发送数据，若客户端没有发送数据，应用程序将会被阻塞直到客户端发送了数据，应用程序才会被唤醒，**read系统调用操作系统从网卡中socket读取客户端发送的数据到内核空间，再把内核空间中的数据复制到用户空间供程序使用。**

    2. BIO（同步阻塞IO）两个阶段都阻塞，**read进行系统调用后应用线程会阻塞直到数据准备好并且从内核空间把数据拷贝到用户空间，是阻塞的**
       BIO中应用程序发起IO的read后会一直阻塞线程直到内核把数据拷贝到用户空间，首先会阻塞系统调用的线程，其次**一个线程只能处理一个SocketIO连接**，大量线程阻塞会导致大量的上下文切换，在高并发下会导致系统资源耗尽
       <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240405195505064.png" alt="image-20240405195505064" style="zoom:50%;" />

    3. NIO（非阻塞IO）read

       - 同步非阻塞 （第一阶段系统调用后非阻塞  第二阶段数据从系统空间拷贝到用户空间阻塞）

         **用户程序通过轮询系统调用read的方式，直到系统将数据准备好，然后阻塞等待系统将数据从系统空间拷贝到用户空间，进程被唤醒处理数据。**
         特点：**一个线程可以处理多个socket连接，但是会对每个连接进行轮询系统调用，不停的从用户态切换到内核态，十分消耗CPU资源**
         <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240406134630688.png" alt="image-20240406134630688" style="zoom:33%;" />
    
       - IO多路复用 **将轮询socket是否就绪交给内核来处理**  
         **调用select/poll是阻塞的，调用epoll_create后还调用epoll_ctl和epoll_wait，read都是阻塞的**
         
         通过IO多路复用，适用于高并发，一个线程可以管理多个连接。
         **程序先发起select/epoll/poll系统调用，将多个socket连接的文件描述符集合放入内核中，让内核轮询socket的数据（避免了应用程序轮询导致的用户态和内核态切换）**，等内核把数据准备好了返回就绪的socket文件描述符给用户态，程序就可以对就绪的socket发起read系统调用，并阻塞直到数据从内核空间拷贝到用户空间。**调用select、poll是阻塞的，直到有socket文件描述符就绪或者超时**
         通过多路复用器，将用户的IO请求注册到多路复用器上，多路复用器轮询到有IO
         <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240405234158637.png" alt="image-20240405234158637" style="zoom:50%;" />
         
       - NIO详解
         
         - Buffer：**NIO读写通过Buffer操作，读操作会将channel中数据写入到buffer中，写操作将buffer中数据写到channel中**，`channel.read(buffer)`,以及其子类都为抽象类，无法通过new实例化，只能通过其静态方法allocate创建实例（实例化的是其非抽象类的子类）`ByteBuffer buffer = ByteBuffer.allocate(1024);`，buffer创建以后默认为写模式,通过flip转换读写模式，get方法读缓冲区数据，put方法读缓冲区数据
         - Channel:channel是双向的，可以将从buffer中的数据写给磁盘或socket ，`channel.write(buffer)`  也可以将socket中数据写到buffer中  channel.read(buffer),每一个channel都对应一个socket
         - Selector:允许一个线程处理多个channel，使用 channel.register(selector,SelectionKey.*OP_ACCEPT*)监听channel的各种事件，selector会不断轮询注册在其中的channel，调用selector.select()可以查看已经就绪的channel，会将就绪的事件加入到就绪集合中，通过selectionKey得到就绪集合，遍历集合进行操作。
         - 如果我们需要使用 NIO 构建网络程序的话，不建议直接使用原生 NIO，编程复杂且功能性太弱，推荐使用一些成熟的**基于 NIO 的网络编程框架比如 Netty**。
         
    
    4. AIO（异步非阻塞线程） 两阶段都是非阻塞的**异步是read后不会阻塞到内核准备好数据  在将socket文件描述符从内核空间拷贝到用户空间也是非阻塞的**
       基于回调实现，**read系统调用后不会阻塞，会在内核中注册一个回调函数**，**内核检测到socket文件句柄就绪后，就会执行回调函数将数据从内核空间拷贝到用户空间，并返回给程序**。
       <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240406135030306.png" alt="image-20240406135030306" style="zoom:50%;" />
       
       ```java
       //使用NIO模拟一个监听8080端口的服务器
       import java.io.IOException;
       import java.net.InetSocketAddress;
       import java.nio.ByteBuffer;
       import java.nio.channels.SelectionKey;
       import java.nio.channels.Selector;
       import java.nio.channels.ServerSocketChannel;
       import java.nio.channels.SocketChannel;
       import java.util.Iterator;
       import java.util.Set;
       public class Tast {
               public static void main(String[] args) {
                   try {
       //                生成一个channel
                       ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
                       serverSocketChannel.configureBlocking(false);
       //                cahnnel的socket方法返回一个与通道相连的scoket，再调用bind方法绑定端口
                       serverSocketChannel.socket().bind(new InetSocketAddress(8080));
       //                  生成一个selector
                       Selector selector = Selector.open();
                       // 将 ServerSocketChannel 注册到 Selector 并监听 OP_ACCEPT 事件
                       serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
       //                  轮询监听事件
                       while (true) {
       //                    select() 等待有事件发生
                           int readyChannels = selector.select();
       
                           if (readyChannels == 0) {
                               continue;
                           }
                           Set<SelectionKey> selectedKeys = selector.selectedKeys();
                           Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
       //                    迭代selectionkeys这个channel集合
                           while (keyIterator.hasNext()) {
                               SelectionKey key = keyIterator.next();
       
                               if (key.isAcceptable()) {
                                   // 处理连接事件
                                   ServerSocketChannel server = (ServerSocketChannel) key.channel();
       //                            accept生成新的socket与客户端通过这个socket进行通信
                                   SocketChannel client = server.accept();
                                   client.configureBlocking(false);
       
                                   // 将客户端通道注册到 Selector 并监听 OP_READ 事件
                                   client.register(selector, SelectionKey.OP_READ);
                               } else if (key.isReadable()) {
                                   // 处理读事件
                                   SocketChannel client = (SocketChannel) key.channel();
       //                            创建一个buffer用来进行缓冲
                                   ByteBuffer buffer = ByteBuffer.allocate(1024);
                                   int bytesRead = client.read(buffer);
       
                                   if (bytesRead > 0) {
                                       buffer.flip();
                                       System.out.println("收到数据：" +new String(buffer.array(), 0, bytesRead));
                                       // 将客户端通道注册到 Selector 并监听 OP_WRITE 事件，写会被触发
                                       client.register(selector, SelectionKey.OP_WRITE);
                                   } else if (bytesRead < 0) {
                                       // 客户端断开连接
                                       client.close();
                                   }
                               } else if (key.isWritable()) {
                                   // 处理写事件
                                   SocketChannel client = (SocketChannel) key.channel();
                                   ByteBuffer buffer = ByteBuffer.wrap("Hello, Client!".getBytes());
                                   client.write(buffer);
                                   // 将客户端通道注册到 Selector 并监听 OP_READ 事件
                                   client.register(selector, SelectionKey.OP_READ);
                               }
                               keyIterator.remove();
                           }
                       }
                   } catch (IOException e) {
                       e.printStackTrace();
                   }
               }
           }
       ```
       
       

​		


#### 5 集合

两大接口Collection和Map。Collection下主要由List、Set、Queue。List下有LinkedList、ArrayList、Vector。Map下有HashMap、TreeMap、HashTable

1. 为什么用集合不用数组？
   创建数组时要声明**长度**，而很多时候数量是不确定的。而且相比于数组，集合可以存储不同类型（泛型）和数量的对象，还有**多种操作**方式（内置方法）。

2. ArrayLIst和数组的区别？（**动态长度，泛型，存放基本类型，只能使用下标操作，Array要指定大小**）
   ArrayList长度是动态的，数组长度固定。
   ArrayList可以使用泛型保证类型安全。数组不可以
   ArrayList只能存对象，而数组可以存基本类型和对象
   ArrayList有丰富的操作入增删改遍历，而数组只能通过下标来访问和修改数据，不能删除数据

3. Vector和Stack和ArrayList
   ArrayList线程不安全（适合于查找比较多的任务），**Vector和Stack线程安全（使用synchronized关键字）**，但是不推荐使用了，可以使用并发集合（concurrentHashmap、copyonwriteArraylist等）。

4. ArrayList可以存放null值但是不推荐，因为不方便维护，可能由于没做判空处理导致空指针异常

5. **当多线程访问ArrayList时，要保证线程安全必须在外部进行同步操作（通常是对ArrayList进行封装）或者直接使用并发集合或者List list=Collections.synchronizedList(new ArrayList)、synchronizedSet、synchronizedMap进行封装**

6. ArrayList和LinkedList区别？（**结构、增删、内存空间**）

   **二者都保存了size，而且LinkedList还保存了链表中最后一个节点**

   - ArrayList使用数组来保存数据，LinkedList使用链表，一个内存是连续的一个是不连续的
   - 插入删除的时间复杂度？ ArrayList插入到最后时间复杂度是O(1),插入到中间时需要移动数据时间复杂度为O(n),LinkedList增删到头尾时因为有指针指向头尾，所以时间复杂度为O(1),当增删到中间时因为没有索引，需要遍历，时间复杂度为O(n)
   - 内存：ArrayList会预留一部分的内存空间，而LinkedList的每个元素要保存前后的指针因此每个元素占用的空间大。
   - ArrayList可以随机访问（即索引），LinkedList只能通过遍历遍历访问
   - 场景：大部分场景下可以用ArrayList而不是LinkedList（因为增删场景中，实际上插入的时候并没有节省时间，要进行遍历元素）

   ArrayList元素
   插入 `add(index,val)`可以指定插入的位置，当从尾部插入复杂度为O(1),否则因为要移动元素，复杂度为O(n)
   删除 remove 的时候从尾部删除复杂度为O(1)，否则复杂度为O(n)
   LinkedList元素在内存上分布是不连续的
   头尾时插入删除复杂度为O(1)，只需要修改头尾节点
   否则因为要遍历节点，复杂度为O(n)

7. **ArrayList扩容？**
   底层使用数组。
   **首先在创建ArrayList时，如果是无参构造默认长度为10，如果有长度按照指定长度创建，参数为包含collection实现类元素则转换为Arraylist。**
   **在add元素的时候会调用ensureCapacityInternal，参数为size+1，判断数组是否需要扩容或者初始化为10，==即需要的size+1大小和实际的大小elementData.length比较==。若大于则调用grow方法进行扩容，如果当前数组长度为0（即第一次add元素），扩容长度为10，否则扩容为1.5倍，如果扩容1.5倍后大于Integer的最大值了，则扩容为Integer最大值**

   - size是Collection和Map的实现类拥有的，也就是集合，length是数组长度，length()是字符串
   - `System.arrayCopy()和Arrays.copyOf()`第一个将原数组上的n个数复制到目标数组的某个位置上（即实现添加元素时的元素后移），第二个调用第一个，将**原数组复制全到一个指定长度的新数组中**（用于扩容）

8. LinkedList为什么不能实现RandomAccess接口？ 
   （由List具体的实现类来实现RandomAccess接口）
   **这个接口中什么都没有定义**，**用于标识一个类是否支持随机访问**，如在binarySearch（二分法搜索某个元素的位置）中会判断是否实现了RandomAccess接口（实现的用索引，没实现的用迭代器）。随机范围接口**即表示该类可以通过索引来访问元素**，没实现的只能使用迭代器访问，因为LinkedList是链表，地址不是连续的所以没法随机访问

9. Comparable和Comparator区别
   两个接口都用于排序。
   Comparable为内部比较器，（集合中元素的类去实现）实现了Comparable的compareTo方法的类的集合可以通过Collections.*sort*()直接进行排序，许多的包装类已经实现了compareable接口。
   Comparator为外部比较器，当类没有实现comparable接口或者实现的compareTo不是我们想要的，可以使用外部比较器，在Collections.sort(list,compatator)排序时传入实现了Comparator接口的比较器实例。

10. HashSet 、LinkedHashMap、TreeMap    无序性 不可重复
    指的是在底层的数组中不是通过索引顺序添加，而是通过hash值来添加
    不可重复是指在添加元素时使用hashcode和equals来判断是否相等

    - HashSet通过hashmap来实现（哈希表），value为一个静态object对象
    - LinkedHashMap在hashMap的基础上加了链表维护插入顺序（哈希表+链表），**（通过accessOrder设置，为true时按照访问顺序进行排序，false则按照插入顺序排序**get访问或者put修改某个元素时，先将该元素删除，再将该元素重新插入链表的尾部**）。**
    - LinkedHashMap **可以用来实现LRU**
      HashMap中遍历是无序的，LinkedHashMap的遍历有序的，在HashMap的基础上，每个节点链表相连，按照插入顺序或者访问顺序进行遍历**,实现LRU时设置参数accessOrder为true与重写removeEldestEntry删除超过设定长度的Entry**
    - TreeMap通过红黑树来实现，**默认用元素的Comparable进行排序**，也可以用Comparator进行排序
    - 性能：HashSet的操作都是O(1)复杂度，除非遇到哈希冲突。LinkedHashSet用链表维护插入顺序，链表迭代会比HashSet慢。TreeSet使用红黑树，操作的时间复杂度为O(logN)

11. HashMap和HashTable
    hashmap不是线程安全的，HashTable是线程安全的但是用力synchronized进行表锁（基本不用），效率很低，当多线程的时候应当使用ConcurrentHashMap
    扩容和初始化：

    - 初始化：不指定大小Hashtable默认大小为11，hashmap默认大小为16
      				指定大小HashTable使用给定大小的哈希表
    - HashMap总是会使用2倍大小的幂作为哈希表的大小（初始化为2的幂，扩容时乘2）。（用tableSizeFor方法保证）
    - 扩容：hashtable每次扩容为2n+1，hashmap扩容为2倍
    - 底层结构：**jdk8后解决哈希冲突的方式为当链表长度大于8时，将链表转化为红黑树（若哈希表长度小于64先进行扩容，而不是转化为红黑树**）

12. HashMap和TreeMap
    TreeMap实现了NavigableMap（与sortMap），sortMap赋予了排序能力（可以自己指定比较器），NavigableMap在排序的基础上可以对集合内元素进行搜索。

    - TreeMap基于红黑树实现，对key进行排序，使用Iterator迭代器遍历时升序输出。可以通过实现外部比较器Comparator来实现排序方式

13. HashSet
    **底层使用HashMap**，如add方法直接调用hashmap的add方法（value值为new Object（））

14. hashmap底层实现

    - jdk1.8之前
      使用数组加链表的方式，当添加键值对的时候，通过key的hashcode计算哈希值，再将hash值与数组长度进行位运算得到在数组中存放的位置，如果当前位置存在数据的话就使用equals进行判断key是否相同，如果相同则val覆盖旧值，如果不存在的话就在链表末尾添加（拉链法） 哈希值的计算方法：在1.8中将hashcode的低位与高位进行异或运算，在1.8之前要进行四次位运算，性能稍差
    - jdk1.8中
      当链表中的个数大于8，且数组长度超过64，就会将链表转化为红黑树，数组长度没超过64则扩容数组。
    - 为什么数组长度为2的幂次方？
      计算键值对放在数组中的位置时，需要将hash值与数组长度进行取模得到数组下标（即hash%n取余），但是当n为2次幂的时候，hash%n ==（n-1）&hash二者相等，长度为2的幂次方时，n-1的二进制数低位全为1，位运算的效率比%效率高。
    - hashmap多线程操作的问题？
      **1.8之前使用头插法，==多个线程同时扩容会导致死循环==，1.8使用尾插法会导致数据覆盖问题**，**==多个线程同时向空桶中插入hash值为一样的数据导致覆盖==**。建议使用concurrenthashmap。
      - 1.8之前扩容会先产生一个新的数组，再将原来数组中每个链表都移动到新的数组中，当多线程执行的过程中两个线程都进行扩容时，可能一个线程刚将执行到移动链表的某处被挂起，线程二同时也进行扩容，并且将该链表扩容完毕，导致线程一在移动链表节点的当前指针和next指针被交换了位置（因为使用头插法，链表的顺序会反过来），从而导致两个节点首尾相连进入死循环。
      - 1.8中多个线程put键值对在hash碰撞的时候可能会造成数据覆盖，当线程一计算完hash值且桶为空的时候被挂起，线程二put的hash值为一样的键值对，桶的第一个节点为线程二的数据，但线程一回来继续执行的时候因为已经判断过hash值与桶为空，会把上一个线程放的值覆盖掉。
    - put流程

      1.判断数组是否为空，为空则初始化数组扩容调用resize
      2.计算key的哈希值，找到在数组的下标，并判断该位置是否有值
      2.1桶中没有值则直接插入
      2.2桶中有值则处理哈希冲突，当桶中第一个元素与put的key相等则直接覆盖
      2.2.1为红黑树，是则放入红黑树中
      2.2.2为链表则遍历进行替换或插入，并判断是否需要转为红黑树或扩容
      3 将size与threshold相比较判断是否需要扩容

    - 1.7==**hashmap扩容**==   **add时会检查是否需要扩容** resize 在数组初始化与扩容会调用resize

      默认大小为16，扩容会将数组大小变为两倍数
      在jdk1.8中，**当链表的长度大于8且数组大小不小于64会将链表转化为红黑树**，否则进行扩容
      负载因子：当map中键值对个数超过了数组长度*负载因子，则进行扩容。**threshold=loadfactor * capacity**。**当size>threshold时进行扩容**
      1.如果阈值已经超过了最大容量，则不进行扩容了
      2.1没有超过最大容量则将数组长度扩充为两倍
      3.创建新数组，将原来数组每个桶中元素都放到新的数组的桶中
      4.移动的方式为先保存当前要移动节点的next节点，再重新计算当前几点在新数组中的位置，并将当前节点的next变为新数组的第一个节点，将新数组第一个节点变为当前节点，再访问之前保存的下一个节点。

      在jdk1.8中对寻找新下标进行了优化，因为下标使用哈希值与数组长度-1进行与的位运算，因此数组长度乘以2，**则新的下标只会在原来位置或者在原来位置再移动二次幂的位置（原来位置+原数组长度）**，就不用重新计算hash了，只要看hash值新增的位是1还是0就好了，且改成了尾插法，杜绝了多线程死循环的问题。

      

15. 迭代器、for each、for、stream

    - 增强for，能在不使用下标的情况下遍历集合或容器，for each遍历EntrySet KeySet  `for (Integer i : map.keySet()`，**实现了Iterable接口的容器都可以使用增强for，底层还是使用Iterator迭代器实现**
    - 迭代器iterator，EntrySet KeySet `Iterator<Map.Entry<Integer,String>> iterator=map.entrySet().iterator();`再`while (iterator.hasNext())`进行迭代`iterator.next()`
    - foreach()方法**是Iterable接口中的一个方法**，所有Collection子类都会实现该Iterable接口，
    - lambda和foreach `map.forEach((k,v)->{System.*out*.println(k+" "+v);});`
    - stream

16. lambda表达式
    为一个匿名函数，jdk8允许把函数作为参数传递给方法  （参数）->{ }

    - **lambda可以代替匿名内部类**只要方法的参数是函数式接口 （如 `new Thread(new Runnable{ 重写的run方法}).start`）变为new `Thread(() -> System.out.println("It's a lambda function!")).start();`，**lambda只用写函数式接口中抽象方法的逻辑**
    - 集合迭代中 `for（String i：list）` 改为 `list.foreach((i)>-{sout(i)})`

17. 函数式接口
    **有且只有一个抽象方法的接口，在作为参数时可以被Lambad表达式或者匿名内部类代替**

18. 快速失败与安全失败
    **在集合Iterator遍历过程中，发现集合中元素被修改的处理机制**，是否允许数据修改

    - 快速失败：即Iterator迭代器遍历时发现数据被修改立刻抛出ConcurrentModificationException异常，导致遍历失败。单线程情况下在迭代中修改数据，或者在多线程情况下线程A在遍历数据的过程中，线程B修改了集合中数据（u**til包下的集合都是快速失败** `Iterator iterator = list.iterator(); Iterator iterator=map.keySet().iterator()`）
    - 失败安全：遍历时修改数据时不会报错因为使用**失败安全时不是访问的原集合，而是对原集合内容的拷贝，所以不会被迭代器检测到，同时也不会遍历到修改的元素**。可以在多线程并发的情况下并发使用与迭代，**concurrent包下的容器都是失败安全的**

19. Iterator迭代器

    [fail-fast（快速失败）机制和fail-safe（安全失败）机制的介绍和区别_fail-fast 与 fail-safe 有什么区别-CSDN博客](https://blog.csdn.net/striner/article/details/86375684)
    Iterator是一个接口（有next，hashnext，remove抽象方法），需要**集合类的内部类去实现Iterator接口**，**调用集合类的iterator()方法会返回一个iterator实现类**，其实现类中有expectedModCount变量（**在创建迭代器时使用modCount赋值**），在每次调用next()方法是时，都会检查modCount集合修改操作的次数是不是等于创建迭代器时修改操作的次数，如果在迭代过程中集合被修改add、remove等操作modcount++则这两个变量值就不相同，就会抛出异常。

    - 解决抛出异常的方式：
      - 使用iterator迭代器中的remove方法，会对exceptModCount重新赋值modCount，就不会报错，删除当前遍历的元素。
      - 使用安全失败，**concurrent包下的容器都是失败安全的，缺点是迭代器无法访问到被修改的数据，因为迭代的是拷贝的集合数据。**
        ![image-20240422173123395](C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240422173123395.png)

20. ConcurrentHashMap

    - 底层结构：jdk1.7中使用数组+链表的方式（使用分段锁），jdk1.8使用数组+链表/红黑树（synchronized与CAS自旋）
    - 线程安全：
      - 1.7中将数组bucket的槽划分为一个个的segment分段锁（把大数组划分成一个个小数组），对每个segment加锁（**每个segment中都有ReentrantLock可重入锁，put时先定位到对应的segement，在调用segment的put方法获取锁，没获取到则自旋，超过一定次数则lock()阻塞获得锁**），每个segment中为entry数组，每个entry又是一个链表，**segment一旦初始化就不能改变**，在要访问某个entry数组中的数据时，要先获得Segment的ReetrantLock锁，对同一个segment的并发操作会被阻塞，不同的segment之间的并发操作不会被阻塞（如put操作先自旋获取锁，超过阈值转为阻塞）
      - 1.8中锁的粒度更细，使用**synchronized+cas自旋的方式，对数组中每个entry加锁 链表时synchronized(节点)，数组该索引位置为空时CAS写入**，只要hashcode不一样就不会阻塞，

    - 1.7put操作：
      - 先计算key的hash值，和segment的长度-1做与运算，得到key所在的segment
      - 获得该位置的segment判断是否为null，如果该segment为空则调用segment的初始化函数，初始化函数先判断该segment是否为null，为null则使用segments[0]容量和负载因子创建一个HashEntry数组，再次检查该位置segment是否为null，避免其他线程创建了，使用创建的数组初始化该segment，自旋检查该位置segment是否为null
      - segment初始化完成，则调用segment.put()方法，segment内部使用可重入锁，先自旋获取锁，计算该key在segment中的下标位置，如果该下标Entry为null直接插入，不为null则当前元素key是否equal，相等则替换，否则遍历链表进行插入
      - 插入完检查是否需要扩容，对单个segment进行扩容，并释放锁
    - 1.8中put，定位bucket的槽，使用自旋put，**如果当前索引为null，则CAS尝试写入，写入成功则break跳出自旋，否则使用synchronized加锁写入数据，最后检查是否需要变成红黑树**
    - **扩容**：**1.7中segment的个数是固定的，每个segment中相当于一个hashmap，因此扩容的时候对segment进行扩容。** 
    - 为什么concurrenthashmap的k，v不能为null？  **多线程时null会存在二义性**。因为在**多线程的时候，数据可能被其他线程修改，而如果返回值为null，就无法判断一个键值对是否被其他线程修改了或者保存的就是null值**，而在hashmap中单线程（只能有一个key为null，val可以为null）不存在其他线程修改数据的问题，null值也就不存在二义性

21. ==扩容机制==

    - ArrayList：使用动态数组，默认容量10，当第一个元素加入才扩容初始化数组，在调用add添加元素时，会检查size+1（即需要的）和elementData数组的（实际长度）的比值，如果需要的长度比实际数组长度大则进行扩容，数组长度扩容为原来大小的1.5倍，`elementData=Arrays.copyOf(elementData,newCapacity)`。创建新数组并拷贝
    - HashMap扩容：1.7中在put中会检测是否需要扩容，当map的size>capacity*threshold时就会扩容，生成一个新的数组，其长度为原来的两倍，再遍历原数组的每个元素所在的链表，计算新的下标位置头插法插入新的数组中，而在1.8对扩容进行了优化，首先1.8中数组的长度都为2的幂次，所以在计算新的下标时只需要判断hash值多的那一位是0还是1，0时则元素的下标不变，1时则将移动到原下标加上数组长度的下标上，第二1.8中对使用尾插法防止在多线程扩容时造成死循环，但是还是会造成数据覆盖等，多线程时应当使用concurrentHashMap
    - ConcurrentHashMap：相比于HashMap，对单个segment进行扩容，使用头插法


#### 6 设计模式

##### 6.1 装饰器

这段代码使用了装饰者模式。在这个模式中，`Decorator` 类扩展了抽象类 `A` 并包含一个 `A` 类型的成员变量，用来引用被装饰的对象。通过在 `Method()` 方法中调用被装饰对象的 `Method()` 方法，然后添加额外的功能，即输出装饰信息，来实现功能的动态扩展。

和代理区别：增加额外的功能是代理，对对象是假控制，装饰器是对功能进行增强，可以进行多层装饰

```java
public class Tast {
    static abstract class A{
            abstract void Method();
    }
    static class B extends A{

        @Override
        void Method() {
            System.out.println("B");
        }
    }
    static class C extends A{
        @Override
        void Method() {
            System.out.println("C");
        }
    }
    static class Decorator extends A{
        A a;
        public Decorator(A a){
            this.a=a;
        }
        @Override
        void Method() {
            a.Method();
            System.out.println("装饰"+a.getClass());
        }
    }
    public static void main(String[] args) {
        B b=new B();
        C c=new C();
        Decorator d1=new Decorator(b);
        d1.Method();
        Decorator d2=new Decorator(c);
        d2.Method();
    }
}
```

##### 6.2适配器

```java
import javafx.scene.shape.Circle;
import java.util.Map;
public class Tast {
    //圆类
    static  class Cricle{
            int radius;
            public Cricle(){}
            public Cricle(int radius){
                this.radius=radius;
            }
             int Radius(){
                 return  radius;
             }
    }
    //正方形类
    static  class Square{
        int width;
        public Square(int width){
            this.width=width;
        }
        int  Width(){
            return  width;
        }
    }
    //一个圆孔 接受的对象为圆
    static class RoundHole{
//        判断圆直径是否能通过
       static Boolean fit(Cricle c){
           return 4-c.Radius()>=0;
       }
    }
//    适配器 需要将方形适配为圆 Square与RoundHole.fit()不想兼容，需要使用适配器
    static class Adaptor extends Cricle {
        Square square;
        public Adaptor(Square square){
            this.square=square;
        }
    @Override
    int Radius() {
        return (int)Math.sqrt(square.width/2);
    }
}
    public static void main(String[] args) {
        Cricle cricle=new Cricle(4);
        Square square=new Square(100);
        Square square1=new Square(5);
        Adaptor adaptor1=new Adaptor(square);
        Adaptor adaptor2=new Adaptor(square1);
        System.out.println("能否通过："+RoundHole.fit(cricle));
        System.out.println("能否通过："+RoundHole.fit(adaptor1));
        System.out.println("能否通过："+RoundHole.fit(adaptor2));
    }
}
```

##### 6.3单例模式    

 懒惰、饥饿、**双重检查锁DCL**、静态内部类、ThreadLocal

[设计模式 - Java中单例模式的6种写法及优缺点对比 - 瘦风 - 博客园 (cnblogs.com)](https://www.cnblogs.com/shoufeng/p/10820964.html)
一个类只有一个实例对象，为该实例提供一个全局的访问节点。
**将默认构造函数设为私有**，防止其他对象使用单例类的new运算符。**新建一个构造方法作为构造函数，该函数会调用构造函数创建对象，并将其保存在一个静态成员变量中**，此后所有对于该函数的调用都会返回该对象，获取对象的唯一方式是调用该类的获取实例的方法。仅在首次请求单利对象时对其进行初始化。工厂模式、生成器模式都可以用单例来实现
场景：如果某个类对于所有客户端都只有一个可用的实例，**无状态的类**

```java
//懒惰模式，在首次调用时才初始化，多线程会有问题
//饥饿模式，直接private static Test test=new Test();在类加载的时候就会对静态变量初始化，是线程安全的，但是会浪费空间
public class Test {
//维护唯一的静态实例    
    private static Test test;
    public String value;
// private构造函数，其他对象无法访问
    private Test(String value) {
        //模拟缓慢初始化
         try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        this.value = value;
    }
//对外暴露静态的构造方法接口
    public static Test getInstance(String value) {
        //首次请求时才会生成实例对象
        if (test == null) {
            test = new Test(value);
        }
        return test;
    }
    public static void main(String[] args) {
         Test test1 = Test.getInstance("asda");
        System.out.println(test1.value); //asda
        Test test2 = Test.getInstance("asda11111");
        System.out.println(test2.value); //asda
    }
}

```

**懒惰模式在多线程环境下要进行特殊处理，即DCL双重检查锁或对getInstance方法加synchronized，避免多个线程多次创建单例对象，多线程可能会同时调用构造方法并获得多个单例的实例

```java
//懒惰模式在多线程下会创造多个单例的实例
    public static void main(String[] args) {
       Thread thread=new Thread(new Runnable() {
           @Override
           public void run() {
               Test aaa = Test.getInstance("aaa");
               System.out.println(aaa.value);
           }
       });
       Thread thread1=new Thread(new Runnable() {
           @Override
           public void run() {
               Test bbb= Test.getInstance("bbb");
               System.out.println(bbb.value);
           }
       });
       thread.start(); //aaa
       thread1.start(); //bbb
    }
```

```java
//双重检查锁，防止多线程下初始化多个单例。
//private static volatile Test test; 要为voiltile对变量的写先于读，主要是为了禁止指令重排序
    public static Test getInstance(String value) {
        //先检查实例是否存在
        if (test != null) {
            return test;
        }
        //加锁
        synchronized (Test.class) {
            //再次判断对象是否存在，因为可能被其他线程获得锁创建了对象
            if (test == null) {
                test = new Test(value);
            }
            return test;
        }
    }
```

静态内部类
外部类加载时不会加载静态内部类，只有在内部类被调用时才会加载内部类

[【Java基础】内部类 （成员内部类、局部内部类、静态内部类、匿名内部类）_局部内部类 匿名内部类-CSDN博客](https://blog.csdn.net/weixin_73442302/article/details/139485714)

```java
public class Test {
//    禁用构造函数
    private Test(){}
    //是线程安全的
    public static Test getInstance(String value) {
        return  Instance.test;
    }
    //内部静态类创建单例对象
    private static class Instance{
        private static Test test=new Test();
    }
    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                Test aaa = Test.getInstance("aaa");
                System.out.println(aaa);
            }
        });
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                Test bbb = Test.getInstance("bbb");
                System.out.println(bbb);
            }
        });
        thread.start();
        thread1.start();
    }
}
```

TheadLocal 性能可能比较低
如果使用反射破坏了单例模式怎么办？反射是调用构造函数来创建实例，可以在构造函数中判断唯一的实例是否已经存在了

```java
private Test(){
	if(test!=null) throw new EXception("此类是单例类，不允许生成新的对象，请通过getInstance获取对象")
}
```

如果被使用clone方法克隆怎么办？重写clone方法，抛出异常

##### 6.4 代理模式

比如需要对第三方包中的某些类进行功能增强（添加日志等），可以使用代理，实现该类同样的接口，在不修改原对象的前提下，提供额外的功能，即通过代理对象来访问目标对象。

- 静态代理

  1、定义接口 2、被代理对象和代理对象都实现相同的接口  3、将目标对象注入代理类，然后通过调用相同的方法来调用目标对象的方法。代理类在调用委托对象的方法前后增加一些操作。

  缺点：不灵活，一旦接口增加了方法，则代理对象和目标对象都需要进行修改。很麻烦，且对每个方法的增强都需要手动完成，对每一个要被代理的类都需要写一个代理类。从JVM角度来说，静态代理在编译时就将接口、实现类、代理类都变成一个个实际的class文件

- 动态代理（JDK） 
  代理对象不用实现接口，目标对象需要实现接口（否则没发用动态代理），所有函数都会经过InvocationHandler的invoke函数转发，因此在invoke中写增加的逻辑。

  Proxy类静态方法newProxyInstance(ClassLoader,Interface[],InvocationHandler)该方法返回值为代理对象，用于生成代理对象。类加载器用于加载代理对象，interface为被代理类所实现的接口，实现了InvocationHandler接口的对象，其中定义了处理逻辑。**调用对象方法时，这个方法调用就会被转发到InvocationHandler中的invoke方法来调用。**

  **步骤**：定义接口及其实现类，自定义InvocationHandler实现类并重写Invoke代理逻辑，在invoke中会调用原生方法与增强逻辑，调用proxy的静态方法newProxyInstance创建代理对象。
  
  ```java
  public interface InvocationHandler {
      /**
       * 当你使用代理对象调用方法的时候实际会调用到这个方法,proxy为动态生成的代理类，method增强的方法，args为方法的参数
       */
      public Object invoke(Object proxy, Method method, Object[] args)
          throws Throwable;
  }
  ```
  
  ```java
  import java.lang.reflect.InvocationHandler;
  import java.lang.reflect.Method;
  import java.lang.reflect.Proxy;
  
  public class EmailProxy {
      public static Object getProxy(Object obj){
          //3、调用proxy的静态方法newProxyInstance创建代理对象。
          return Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj.getClass().getInterfaces(), new InvocationHandler() {
              //2、自定义InvocationHandler实现类并重写Invoke代理逻辑
              @Override
              //invoke函数接收被代理对象proxy，并且调用其原生方法method并完成代理逻辑
              public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                  System.out.println("执行前代理逻辑");
                  Object res=method.invoke(obj,args);
                  System.out.println("执行后代理逻辑");
                  return res;
              }
          });
      }
  
      public static void main(String[] args) {
          //1、定义EmailInter接口及其实现类EmailImpl
          EmailInter proxy = (EmailInter) getProxy(new EmailImpl());
          proxy.send();
      }
  }
  ```
  
- Cglib代理
  **动态代理只能代理实现了接口的类**，并且只能代理接口中实现的方法，而被代理类自己的方法，无法被代理调用。**Cglib在运行时对字节码进行修改和动态生成构建一个子类对象**，实现对目标对象功能的扩展，因为是生成一个子类对象，所以没法代理声明为final的方法和类。
  
  步骤：0、先手动添加第三方依赖 1、定义要被代理的类 2、自定义实现MethodInterceptor并重写intercept方法（类似于JDK中InvocationHandler的invoke方法）3、通过Enhance类的create创建代理类。 
  
  ```java
  public interface MethodInterceptor
  extends Callback{
      // 拦截被代理类中的方法,obj为被代理对象、method需要增强的方法，args为方法参数、proxy用于调用原始方法
      public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,MethodProxy proxy) throws Throwable;
  }
  //Enhance先设置类加载器、被代理类、拦截器之后再创建代理类对象
  import net.sf.cglib.proxy.Enhancer;
  public class CglibProxyFactory {
  
      public static Object getProxy(Class<?> clazz) {
          // 创建动态代理增强类
          Enhancer enhancer = new Enhancer();
          // 设置类加载器
          enhancer.setClassLoader(clazz.getClassLoader());
          // 设置被代理类
          enhancer.setSuperclass(clazz);
          // 设置方法拦截器
          enhancer.setCallback(new DebugMethodInterceptor());
          // 创建代理类
          return enhancer.create();
      }
  }
  
  ```
  

#### 8 多线程

##### 8.1 ArrayList并发插入数据的问题

[java多线程之 几种线程安全的list synchronizedList()&CopyOnWriteArrayList_java synchronizedlist-CSDN博客](https://blog.csdn.net/zhaofuqiangmycomm/article/details/113102649)
会遇到**数组越界报错**或**实际插入个数少于希望插入个数**的问题，插入add时先检查插入后size+1集合大小是否足够，不够则进行扩容，下一步才插入元素并将size++，比如数组大小为10，目前有9个元素，当线程1检查完size+1==10后时间片用完，线程2也检查集合大小并插入元素，此时数组大小变为10了，线程1得到时间片继续进行插入元素，此时数组已经满了，就会报错越界。也有可能当数组没满时，因为size不是volatile修饰的，线程1得到size为5，线程2也得到size为5，线程1插入size++位置完成后，此时线程2的size还是5，就有可能将线程1在位置6的插入的元素覆盖，导致最后插入的个数少于希望的个数

线程安全的ArrayList：对**操作方法加锁**，使用**Collections.synchronizedList(list)获得线程安全的容器**会对每个读写操作加锁、使用**CopyOnWriteArrayList**读读不互斥，读写也不互斥，add时先获取Reentrantlock，得到旧数组并复制到len+1的新数组中，插入到新数组中，最后再将新数组替换老数组，最后释放锁，因为add操作不是直接在原数组中完成，所以读写并不互斥，数组用volatile修饰保证修改的可见性。**写加锁并不在原数组修改，读不加锁，对数组用volatile**，相比Collections.synchronizedList(list)读写都加锁，性能很高，且很实用与读多写少的场景
<img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240422154056309.png" alt="image-20240422154056309" style="zoom:50%;" />

1. 为什么要多线程？

   一个cpu中有多个核心，线程是核心处理的单位，一个线程同一时刻只能运行在一个核心上，如果是单线程那么同一时刻只有一个核心会被占用，其他的核心都会被浪费，无法提升执行效率。因此利用多线程并行执行这些线程就可以发挥多核的优势，让程序执行的更快。

2. 线程和进程

   - 进程是分配资源的最小单位（程序运行的基本单位）比如启动main函数会开启一个进程，main所在的线程为主线程

   - 线程是调度资源的最小单位（一个进程可以有多个线程，线程之间共享进程的堆和方法区的资源），线程独有程序计数器，栈，本地方法栈，所以线程之间切换负担比切换进程要小的多。
   - 一个java线程对应一个系统内核线程。

3. 并行与并发
   多个线程再同一个核上交替执行，为并发（同一时间段执行）

   多个线程再多个核上同时执行，为并发（同一时刻执行）
   <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240507165956312.png" alt="image-20240507165956312" style="zoom:25%;" />

4. 创建线程
   [大家都说Java有三种创建线程的方式！并发编程中的惊天骗局！ (qq.com)](https://mp.weixin.qq.com/s/NspUsyhEmKnJ-4OprRFp9g)

   - 继承Thread类、实现Runnable接口、实现Callable接口、线程池工具类Executors或ThreadPoolExecutor、new Thread().start()创建。

5. 线程状态

   [Java面试必会：线程如何在 6 种状态之间转换？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/130875882)
   NEW, RUNNABLE（包含running、ready）, BIOCKED, WAITING, TIME WAITING, TERMINATED。
   阻塞：

   - waiting（线程中执行wait()、Lock失败、其他线程join），JVM将当前线程放进等待队列要有条件如join执行完、notify、unlock主动唤醒。
   - blocked（notify后没拿到monitor锁、synchronized获得monitor锁失败）。
     <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240507152052021.png" alt="image-20240507152052021" style="zoom:67%;" />

6. sleep和wait区别
   - sleep()没有释放锁，而wait()释放了锁。
   - sleep用于暂停执行，wait用于线程交互通信。
   - wait被调用，线程不会自动苏醒，需要notify唤醒并争夺锁。sleep时间到了自动苏醒
   - sleep是线程的静态方法。wait是Object类的本地方法。

7. 可以直接调用Thread类的run方法吗？
   不能。Thread只有调用start方法才会新建一个线程并进入runnable状态，并自动执行run方法的内容。直接运行run方法会把run方法当成main线程下的一个普通方法执行，并不会单独创建一个线程，所以不是多线程工作。

8. 重排序
   重排序可能会导致可见性的问题，使用volatile或锁解决。重排序的目的是使用多线程提高执行性能
   - 编译器重排序，在不改变单线程语意的情况下改变语句执行顺序
   - 指令重排序， 如果不存在数据依赖，处理器可以改变语句对应指令的执行顺序
   - 内存系统重排序，处理器使用缓存和读写缓冲区的多级缓存，使得加载和存储可能是乱序的

9. volatile
   可以保证变量的可见性，但无法保证执行的原子性（如i+=1由三个指令组成，但是并不是原子性的）。使用synchronized、AutomaicInteger原子类、ReentrantLock来保证原子性
   [Java并发常见面试题总结（中） | JavaGuide](https://javaguide.cn/java/concurrent/java-concurrent-questions-02.html#volatile-可以保证原子性么)

10. 悲观锁和乐观锁

    - 悲观锁每次访问资源都先加锁再访问，共享资源每次只有一个线程能访问，其他线程阻塞。如**synchronized和ReentrantLock。**
    - 乐观锁每次访问数据时，**假设在操作期间旧值没有发生改变**，若没有变化则更新新值，否则不更新新值。如使用版本号或者CAS。**automic包下的原子变量类使用的就是乐观锁**。先获得版本号，拷贝一份数据进行修改，比较版本号若版本号没变，则将修改提交（比较并修改是一个原子性操作，使用unsafe进行操作系统的本地调用实现的）
    - 对比：高并发下悲观锁会造成线程阻塞与频繁的上下文切换，还有可能造成死锁。乐观锁不会造成死锁也不会线程阻塞，但是频繁的冲突也会造成性能的下降。悲观锁适用于写比较多（竞争激烈），乐观锁适用于写比较少（竞争少），避免频繁的加锁。

##### 8.2 多线程代码

1. 500个线程同时对count进行自增操作，其结果可能会小于500.
   原因：可见性，一个线程A对变量i的修改，另一个线程B没有立即看到，线程A对变量i的修改只保存在了线程的本地内存中，而并没有立即写到主内存中，而线程B从主内存中获取该的变量即为旧值。原子性，一个操作要么全一次性执行完要么都不执行，如i+=1为三个指令，cpu从内存读i，cpu执行i+1，cpu将结果写回内存，由于cpu执行是分片的，如线程A和B都执行i+1，可能线程A执行完指令1，线程B开始执行这个三个指令，就会导致i的结果不为3而为2，就是本代码结果小于500的原因。

   <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240507163725591.png" alt="image-20240507163725591" style="zoom:67%;" />


   ```java
   public class Test {
       public static void main(String[] args) throws InterruptedException {
           final int size=500;
           final CountDownLatch countDown=new CountDownLatch(size);
           ExecutorService executorService = Executors.newCachedThreadPool(); //无线创建新线程
           Example example = new Example();
           for (int i=0;i<size;i++){
               executorService.execute(()->{
                   example.add();
                   countDown.countDown();
               });
           }
           countDown.await(); //阻塞当前线程直到countDown为0时当前线程才继续执行，即保证循环了500次
           executorService.shutdown();
           System.out.println(example.getCount());
       }
       static class Example{
           private int count=0;
           public void add(){
               count++;
           }
           public int getCount(){
               return count;
           }
       }
   }
   ```

   2.5个线程同时对i进行500次自增操作，volatile只能保证可见性，但不能保证指令执行的原子性
```java
public class Test {
    volatile int i=0;
    public  void inc(){  //若该方法加上synchronized则可以保证指令执行的原子性
        i++;
    }
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        Test test = new Test();
        for(int i=0;i<5;i++){
            executorService.execute(()->{
                for(int j=0;j<500;j++){
                    test.inc();
                }
            });
        }
        Thread.sleep(1500);
        executorService.shutdown();
        System.out.println(test.i);  //2296而不是2500
    }
}
```



#### 7.代码

##### 7.1 linkedHashMap

可以用来实现LRU

```java
 public static void main(String[] args) {
        LinkedHashMap<String, String> accessOrderedMap = new LinkedHashMap<String, String>(16, 0.75F, true){
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, String> eldest) { // 实现自定义删除策略，否则行为就和普遍Map没有区别
                return size() > 3;
            }
        };
        accessOrderedMap.put("Project1", "Valhalla");
        accessOrderedMap.put("Project2", "Panama");
        accessOrderedMap.put("Project3", "Loom");
        accessOrderedMap.forEach( (k,v) -> {
            System.out.println(k +":" + v);
        });
        // 模拟访问
        accessOrderedMap.get("Project2");
        accessOrderedMap.get("Project1");
        accessOrderedMap.get("Project3");
        System.out.println("Iterate over should be affected:");
        accessOrderedMap.forEach( (k,v) -> {
            System.out.println(k +":" + v);
        });
        // 触发删除
        accessOrderedMap.put("Project4", "Mission Control");
        System.out.println("Oldest entry should be removed:");
        accessOrderedMap.forEach( (k,v) -> {// 遍历顺序不变
            System.out.println(k +":" + v);
        });
    }
```

##### 7.2 优先级队列priorityqueu

##### 7.3 LRU

```java
class LRUCache {
    // 链表+Map 类似于LinkedHashMap
    class Node{
        int key;
        int value;
        Node pre;
        Node next;
        public Node(int key,int value){
            this.key=key;
            this.value=value;
        }
        public Node(){}
    }
    HashMap<Integer,Node> map;
    int capacity;
    int size;
    Node head;
    Node tail;
    public LRUCache(int capacity) {
        map=new HashMap<>();
        head=new Node(-1,-1);
        tail=new Node(-1,-1);
        head.next=tail;
        tail.pre=head;
        this.capacity=capacity;
        this.size=0;
    }
    
    public int get(int key) {
        Node res=map.get(key);
        if(res==null) return -1;
        // 将节点移到头部
        moveToHead(res);
        return res.value;
    }
    
    public void put(int key, int value) {
        Node node=map.get(key);
        Node temp=new Node(key,value);
        //是否已经存在 存在则删除重新插入
        if(node!=null){
            deleteNode(node);
            addToHead(temp);
        }else{
            //不存在则插入
            addToHead(temp);
        }
        //判断大小
        if(size>capacity){
            deleteNode(tail.pre);
        }
    }
    private void deleteNode(Node temp){
        size--;
        map.remove(temp.key);
        temp.pre.next=temp.next;
        temp.next.pre=temp.pre;
    }
    private void addToHead(Node temp){
        temp.next=head.next;
        head.next=temp;
        temp.next.pre=temp;
        temp.pre=head;
        size++;
        map.put(temp.key,temp);
    }
    private void moveToHead(Node temp){
        deleteNode(temp);
        addToHead(temp);
    }
}
```





















1. 本地方法栈
2. 堆
3. 方法区