# 第一章 走进Java
- JDK是用于支持Java程序开发的最小环境
- JRE是支持Java程序运行的标准环境
- JDK7是可以支持Windows XP操作系统的最后一个版本
- Oracle再迫使商业用户要么不断升级JDK的版本，要么就去购买商业支持
# 第二章 内存区域与内存溢出异常
![avator](img/1.jpg)
## 程序计数器
<p>程序计数器（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。</p>

<p>在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。</p>

## Java虚拟机栈
<p>Java虚拟机栈（Java Vitrual Machine Stack）也是私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候，Java虚拟机同步创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每一个方法被调用直至执行完毕的过程，就对应一个栈帧在虚拟机栈中从入栈道出栈的过程。</p>
<p>虚拟机栈中的局部变量表存放了编译期可知的各种Java虚拟机基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它并不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄会这其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令地址）。</p>
<p>这些数据类型在局部变量表中的存储空间以局部变量槽（Slot）来表示，其中64位长度的long和double类型的数据会占用两个变量槽，其余的数据类型只占用一个。局部变量表所需的内存空间在编译期间完成分配。在方法运行期间不会改变局部变量表的大小。</p>
<p>如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果Java虚拟机栈容量可以动态扩展，当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError</p>

## 本地方法栈
<p>本地方法栈是为虚拟机使用到的本地（Native）方法服务</p>
<p>本地方法栈也会在栈深度溢出或栈扩展失败是分别抛出StackOverError和OutOfMemoryError</p>

## Java堆
<p>Java堆是被所有线程共享的一块内存区域</p>
<p>Java堆是垃圾收集器管理的内存区域，因此一些资料中它也被称作“GC堆”</p>
<p>Java堆可以处于物理上不连续的内存空间中，但它逻辑上应该被视为是连续的</p>
<p>Java堆中没有内存完成实力分配，并且堆也无法再扩展时，Java虚拟机将会抛出OutOfMemoryError</p>

## 方法区
<p>方法区（Method Area）是各个线程共享的内存区域，它用于存储已被虚拟机加载的类型信息、常量、静态变量、即使编译器编译后的代码缓存等数据</p>
<p>原则上如何实现方法区属于虚拟机实现细节，不受《Java虚拟机规范》管束，并不要求统一</p>
<p>根据Java虚拟机规范的规定，如果方法区无法满足新的内存分配需求时，将抛出OutOfMemoryError异常</p>

## 运行时常量池
<p>运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），用于存放编译器生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常<p>
## 直接内存
<p>一般配置虚拟机参数时，会根据实际内存去设置-Xmx等参数信息，但经常忽略掉直接内存，使得各个内存区域的总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常</p>

## 对象的创建
<p>当Java虚拟机遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程</p>
<p>在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需内存的大小在类加载完成后便可完全确定，为对象分配空间的任务实际上便等同于把一块确定大小的内存块从Java堆中划分出来</p>
<p>对象创建在虚拟机中是非常频繁的行为，即使仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的。解决这个问题有两种可选方案：一种是对分配内存空间的动作进行同步处理——实际虚拟机是采用CAS配上失败重试的方法保证更新操作的原子性；另一种是把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB），哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完了，分配新的缓存区时才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX:/-UseTLAB参数来设定。</p>
<p>内存分配完成之后，虚拟机必须将分配到的内存空间（但不包括对象头）都初始化为了零值。接下来，Java虚拟机还要对对象进行必要的设置。</p>

## 对象的内存布局
<p>在HotSpot虚拟机里，对象在堆内存中的存储布局可以划分为三个部分，对象头（Header），实例数据（Instance Data）和对象填充（Padding）。</p>
<p>HotSpot虚拟机对象的对象头部分包括两类信息。第一类是用于存储对象自身的运行时数据，如哈希吗（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32个比特和64个比特，官方称它为“Mark Word”</p>

![avator](img/2.jpg)

<p>对象头的另一部分是类型指针，即对象指向它的类型元素数据的指针，Java虚拟机通过这个指针来确定该对象是哪个类型的实力。</p>
<p>实例数据部分是对象真正存储的有效信息。类字段的存储顺序会受到虚拟机分配策略参数（-XX:FieldsAllocationStyle参数）和字段在Java源码中定义顺序的影响。HotSpot虚拟机默认的分配顺序为longs/doubles、ints、shorts、chars、bytes/booleans、oops（Ordinary Object Pointers，OOPs）。在默认情况下，父类中定义的变量会出现在子类之前。如果HotSpot虚拟机的+XX:CompactFields参数值为true（默认就为true），那子类之中较窄的变量也允许插入父类变量的空隙之中，以节省出一点点空间。</p>
<p>对象的第三部分是对齐填充，它仅仅起着占位符的作用。由于HotSpot虚拟机的内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是任何对象的大小都必须是8字节的整数倍。对象头部分已经被精心设计成正好是8字节的倍数（1倍或者2倍），因此，如果对象实例数据部分没有对齐的话，就需要通过对齐填充来补全。</p>

## 对象的访问定位
<p>对象访问方式也是由虚拟机实现而定的，主流的访问方式主要有使用句柄和直接指针两种。</p>
<p>使用具柄来访问的最大好处就是reference中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要被修改。</p>
<p>使用直接指针来访问的最大好处就是速度快，它节省了一次指针定位的时间开销。</p>

## Java堆溢出

略

## Java栈溢出

略

## 方法区和运行时常量池溢出

在JDK6或更早之前的HotSpot虚拟机中，常量池都是分配在永久代中，我们可以通过-XX:PermSize和-XX:MaxPermSize限制永久代的大小。JDK7继续使用-XX:MaxPermSize限制方法区大小，JDK1.8使用-XX:MaxMetaspaceSize。但是自JDK1.7开始，原本存放在永久代的字符串常量池被移至Java堆中。

在经常运行时生成大量动态代理类的应用场景里，就应该特别注意这些类的回收情况。这类场景除了CGLib字节码增加和动态语言外，常见的还有：大量JSP或动态产生JSP文件的应用、基于OSGi的应用等(下面代码片段会出现java.lang.OutOfMemoryError: PerGen space(JDK1.7版本))。

```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

/**
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMethodHodAreaOOM{
    public static void main(String[] args){
        while(true){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    return methodProxy.invokeSuper(o, objects);
                }
            });
        }
    }
    static class OOMObject{

    }
}
```

JDK1.8之后，永久代完全被元空间替代，JVM参数如下：

- -XX:MaxMetaspaceSize：设置元空间最大值，默认-1，即不限制或者说收本地内存大小限制
- -XX:MaxspaceSize：元空间的初始值，单位字节。会根据垃圾收集的类型卸载情况自动调节这个值（不超过最大设置值）
- -XX:MinMetaspaceFreeRatio：垃圾收集之后控制最小元空间的百分比
- -XX:MaxMetaspaceFreeRatio：垃圾收集之后控制最大元空间的百分比

## 本机直接内存溢出

直接内存（Direct Memory）的容量大小可通过-XX:MaxDirectMemorySize参数来指定，如果不去指定，则默认与Java堆最大值（-Xmx指定）一致。

直接内存导致的内存溢出，一个明显的特征书在Heap Dump文件中不会看见有什么明显的异常情况，如果读者发现内存溢出之后产生的Dump文件很小，而程序中有直接或者间接用了DirectMemory（如NIO），那就可以考虑重点检查一下直接内存方面的原因了。

# 第三章 垃圾收集器与内存分配策略

## 可达性分析算法

可达性分析（Reachability Analysis）算法的基本思路就是通过一系列成为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径成为“应用链”（Reference Chain），如果某个对象到GC Roots间没有任何引用链相连，或者用图论的话来说就是从GC Roots到这个对象是不可达时，则证明此对象是不可能再被使用的。

![avator](img/3.png)

<p></p>

## 再谈引用

JDK1.2版本之前，Java里面的引用是很传统的定义：如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称该refercence数据是代表某块内存、某个对象的引用。

在JDK1.2版本之后，Java对应用的概念进行了扩充，将引用分为强秦勇（Strongly Reference）、软引用（Soft Reference）、弱引用（Weak Refercence）和虚引用（Phantom Reference）4种。

- 强引用是最传统的“引用”定义，如“Object obj = new Object()”这种引用关系。只要强引用关系还存在，垃圾收集器就永远不会回收掉引用的对象。
- 软引用是用来描述一些还有用，但非必要的对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收。在JDK1.2版本之后提供了SoftReference类来实现软引用。如缓存类型的数据，可以使用软引用进行定义，在内存不足的时候进行回收，再合适的升级再重新load。

```java
import java.lang.ref.SoftReference;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class SoftReferenceMap{
    public static void main(String[] args){
        SoftReference<Map<String, String>> softReference = new SoftReference<Map<String, String>>(new HashMap<String, String>());
        Map<String, String> map = softReference.get();
        if (map == null) {
            softReference = new SoftReference<Map<String, String>>(
                    map = new HashMap<String, String>());
        }
    }
}
```

- 弱引用也是用来描述哪些非必要的对象，但是它的强度比软引用更弱一些，被软引用关联的对象只能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉被弱应用关联的对象。在JDK1.2之后提供了WeakReference类来实现弱引用。

- 虚引用是最弱的一种引用关系。无法通过虚引用获取一个对象的实例。为一个对象设置虚引用关系的唯一目的是为了能在这个对象被收集器回收时收到一个系统通知。在JDK1.2版本之后提供了PhantomReference类实现虚引用。虚引用必须配合ReferenceQueue使用，当垃圾回收器决定对PhantomReference对象进行回收时，会将其插入ReferenceQueue中。在NIO直接内存回收时，会使用到虚引用

## 对象存活判定

1. 可达性分析算法判断对象为不可达的，对对象进行第一次标记
2. 此兑现是否有必要执行finalize()方法，如果没有覆盖finalize()方法或者已被虚拟机执行，则判定为“没必要执行”
3. 如果有必要执行finalize()方法，那么该对象会被放到一个名为F-Queue的队列中，稍后由一个低优先级的Finalizer线程去执行他们的finalize()方法。但虚拟机并不保证他们能够执行完成，防止finalize()方法执行缓慢或死循环而导致F-Queue队列中的其他对象长时间或永久处于等待状态
4. 稍后收集器将对F-Queue中的对象进行第二次小规模的标记。在finalize()方法中只要对象重新与引用链是哪个的任何一个对象建立关联它就可以继续存活（不建议使用finalize()方法）

## 回收方法区

方法区的垃圾收集的“性价比”通畅比较低。方法区的垃圾收集主要回收两部分内容：废弃的常量和不再使用的类型。

废弃的常量：如“java”曾进入常量池中，但是当前系统又没有任何一个字符串对象值是“java”，也就是说没有任何字符串对象引用常量池中的“java”常量。如果此时发生内存回收，并且收集器判断有必要的话，“java”会被系统清理出常量池。

类型不再使用的条件：

- 该类的所有实例都已经被回收
- 加载该类的类加载器已经被回收，这个条件很难达成
- 该类对应的java.lang.Class对象没有在任何地方被引用

只有满足上面三个条件，在被允许回收，而不是必然被回收。关于是否要对类型进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制，还可以使用-verbose:class以及-XX:+TraceClassLoading、-XX:+TraceClassUnLoading查看类加载和卸载信息。

## 分代收集理论

1）弱分代加锁（Weak Generational Hypothesis）：绝大多数对象都是朝生夕灭的

2）强分代加锁（Strong Generation Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡

这两个分代加锁共同奠定了多款常用的垃圾收集器的一致的设计原色：收集器应该讲Java堆划分出不同的区域，然后将回收对象依据其年龄（年龄即对象熬过垃圾收集过程的次数）分配到不同的区域之中存储。

分代收集理论会把Java堆划分为新生代（Young Generation）和老年代（Old Generation）两个区域。在新生代中，每次垃圾收集时都发现有大批对象死去，而每次回收后存货的少量对象，将会逐步晋升到老年代中存放。

值得注意的是，新生代对象完全有可能被老年代所引用，反过来同样如此。这就引出第三假说：

3）跨代应用假说（Intergenerational Reference Hypothesis）：跨代引用相对于同代引用来说仅占极少数。也就是说存在互相引用关系的两个对象，是应该倾向于同事生存或同事消亡的。

所以为了解决跨代引用，只需在新生代上建立一个全局的数据结构（该结构被称为“记忆集”，Remembered Set），这个结果把老年代划分成若干小块，标识出老年代那一块内存会存在跨代引用。此后当发生Minor GC时，只有包含了跨代引用的小块内存的对象才会被加入到GC Roots进行扫描。

```html
部分收集Partial GC：指目标不是完整收集整个Java堆的垃圾收集，其中有包括：
	新生代收集Minor GC/Young GC：目标是新生代的垃圾收集
	老年代收集Major GC/Old GC：目标是老年代的垃圾收集。目前只有CMS收集器会有单独收集老年代的行为
	混合收集Mixed GC：目标是收集整个新生代以及部分来年代的垃圾收集
整堆收集Full GC：收集整个Java堆和方法区的垃圾收集
```

## 标记-清除算法

标记-清除（Mark-Sweep）算法分为“标记”和“清除”两个阶段：首先标记处所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象，也可以反过来，标记存活的对象，统一回收所有未被标记的对象。标记过程就是对象是否属于垃圾的判定过程。

它的主要缺点有两个：第一是执行效率不稳定，如果Java对中包含大量对象，而且其中大部分是需要被回收的，这时需要进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而减低；第二个是内存空间的碎片化问题，标记、清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前出发另一次垃圾收集动作。

![avator](img/4.png)

## 标记-复制算法

标记-复制算法常被称为复制算法。它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就讲还存货的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。

它的优点是实现简单，运行高效。缺点是将可用内存缩小为原来的一半。

由于对象是朝生夕灭的，所以提出另一种更为优化的复制算法，成为Appel式回收。

具体做法是把新生代分为一块较大的Eden空间和两块较小的Survivor空间，每次分配内存只使用Eden和其中一块Survivor。发生垃圾收集时，将Eden和Survivor中仍然存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden和已用过的那块Survivor空间。HotSpot虚拟机默认Eden和Suvivor的大小比例是8：1，也即每次新生代中可用内存空间为整个新生代容量的90%（Eden的80%加上一个Survivor的10%），只有一个Survivor空间，即10%的新生代是会被“浪费”的。

## 标记-整理算法

“标记-整理”（Mark-Compact）算法，其中的标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向内存空间的一端移动，然后直接清理掉边界以外的内存。

如果移动存活对象，尤其是在老年代这种每次回收都有大量对象存活的区域，移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作，而且这种对象移动操作必须全程暂停用户应用程序才能进行

