## Java内存模型

### 原子性
一个操作为原子性的重要依据就是此操作是不能被中断的，即使是在多线程并行执行的时候，一旦操作开始，就不会被其他线程干扰。
JVM中的原子操作包括：
除long以及double外的所有基础类型的赋值操作。
引用类型的对象赋值。
所有java.concurrent.Atomic包中的类的所有操作。
加上了volatile修饰符的long与double的赋值操作。注意volatile关键字只能保证可见性，并不能保证原子性。这里所说的赋值操作只是针对与单纯的get和set操作，并不适用与getAndOperate操作。volatile修饰的变量在经过编译器编译成字节码的时候，会在其load（get操作）之前增加一个内存屏障指令，也会在其store（set操作）后增加一个内存屏障指令。这个屏障存在的目的为了实现volatile的可见性。此内存屏障指令存在的目的是：
1、使部分特定的指令执行的顺序得到一个保证。
2、影响一些数据的可见性。
由于为了优化指令的执行效率，编译器会对指令进行重排序。当重排序遇到了此内存屏障指令，相当于告诉cpu，此指令之前的指令必须先执行，后于此指令的必须后执行。同样执行此指令的时候会强制进行一次不同cpu执行的线程的工作内存的刷新。
volatile所添加的两个内存屏障的作用就是:
1、在load操作之前加入内存屏障指令是为了刷新此线程的工作内存为主存中的最新值，保证之前发生过的事情已经发生，可以看见任何更新过的值。
2、在store操作之后加入内存屏障指令是为了将此线程在工作内存中修改的变量值提交的主存，以至于在完成写入操作后，所有线程都能看到最新的值。
那么为啥volatile只能保证get/set的原子性，并不能保障其他操作的原子性，如getAndOperate操作。这里对一个经过volatile修饰的int型数据进行自增操作。在经过编译器转换为字节码后可以大致分为三步：
1、将volatile变量读取到工作内存
2、增加变量的值
3、将工作内存中的变量写回，让其他线程可见。其详细的字节码可以总结为下述四条：
load
increme
store
barrier
此时若存在多个线程执行这个自增操作，尽管最后一步内存屏障将数据同步到主存，使其他cpu可获得最新的值，但其上的三条指令（从load到store）都是不安全的。在这种情形下，可能会存在某个线程的修改值在提交时，将其他线程的修改值改抹掉，使修改值丢失。 


### 重排序
编译器生成的字节码指令的次序，可以与其源代码所指明的顺序是不同的，这种操作是出于对指令执行的优化以及对全局寄存器进行一个更好的分配与使用，以提高指令执行的效率。
重排序的类型：
指令可以乱序执行，甚至并行执行。
指令的生成序列可以与其源代码指明的表面顺序不同。

并发与并行：
并发和并行都可是多线程架构模式下，支持多线程。但其不同的是，如果多个CPU可以在同一时刻执行多个线程，这个情形就被叫做并行。否则若CPU在同一时刻只能执行一个线程，多个线程排队等待执行，此种情况就叫做并发。

### 可见性
在多线程&多处理器架构下，有两个同时执行的A、B线程（并发并行）、此时A线程可能无法立即看到线程B操作的结果。所以Java内存模型规定了JVM的一种保证：什么时候来写入一个变量进入主存是对其他线程可见的。
上述情形发生的原因：
在Java内存模型规定所有的变量都存储在主内存中，但对于其线程来说，每条线程都拥有自己的工作内存，此工作内存中包含的变量都是来源于主内存中变量的拷贝，而线程中对其变量的操作，如赋值、读取等都是发生在其工作内存中，而不是直接对其主内存的值。不同的线程之间无法直接访问互相的工作内存，线程间的变量值传递只能间接通过主内存来完成。所有当某个线程没有将修改值及时同步到主存或者没有及时从主存中同步最新值时，就会发生上述情形。

几种可能发生上述情形的情况：
A线程修改的变量值存放在寄存器当中。
A线程修改的变量值存放在高速缓存之中。
B线程使用的依旧是A线程修改前的主存值的拷贝，没有及时同步拉起主存的最新值。

可见性的概念：当一个变量在多个线程中拥有其在主内存中的数据副本，此时如果一个线程A对其包含的数据副本进行了修改，那么其他线程也应该看到这个修改后的值。

### happens-before
happens-before的关系保证：A（线程或操作）与B（线程或操作）若满足此关系，此时A的执行结果对于B是内存可见的。如果两个操作没有进行happens-before保证，JVM对其进行重排序将是任意的。

几种happens-before的情形：
在某个程序中（代码中），若所有的A操作都发生在B操作之前，就可以保证每一个A都happens-before与每一个B操作。
对于使用synchronized修饰时，其在Java字节码中其实对其修饰代码段或者方法或者对象给加上了一个监视器monitor。在对某个监视器进行解锁操作都是happens-before其加锁操作。
对于volatile对象的写操作都是happens-before读操作。
happens-before的传递性：A hb B、B hb C，那么A hb C。

### 线程、工作内存、主内存关系

所有线程共享主内存，但其进行执行时，使用的变量数据是来自于其自身的工作内存。而工作内存中的变量其实是来自于主内存中的一个副本拷贝。

线程的工作内存其实是寄存器和高速缓存的一个抽象描述，线程消耗的硬件资源其实主要是CPU，而CPU在计算时所读取的资源并不是都来自于主内存中（由于访问主内存的速率问题），而是引入了寄存器和高速缓存这两级内存单元，并按照寄存器-》高速缓存-》主内存优先级来存放变量数据的。线程中使用的工作内存（即寄存器和高速缓存）中的变量数据都是来自主内存中的拷贝。在多线程并行执行时，多个线程拥有自己独立的工作内存，在线程执行过程中，可能有多个线程对相同的变量进行了修改，但由于线程之间不存在直接的数据同步操作，所有这些线程互相是不知道其他线程以及修改了这个变量的值，这就是**可见性**的由来。这也是JAVA内存模型的要解决的几个特性：原子性、有序性、可见性以及happens-before。
JVM是一个虚拟的计算机，其也会遇到上诉问题，但由于其不能直接控制线程对寄存器和高速缓存的同步，所以在语法上提供了一种解决方案：synchronized，volatile和各种锁机制。

线程安全的基本要求是此线程锁管理的变量数据是否能安全的被其他线程进行访问。
