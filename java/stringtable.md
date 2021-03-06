# jvm连载：内存结构
## tips
* jvm不像代码那样对程序员很直观，基础概念和理论又特别多
* 本着理解记忆，了解每块概念，多看多计
* 着重理解jvm运行时数据区，每个区域的作用，是否线程私有，有什么异常

## 程序计数器
* jvm在执行指令时需要知道要执行哪条指令，计数器存储的就是下一条要执行的指令地址
* 具体的工作过程：解释器在解析jvm指令时，根据计数器的地址依次执行指令，将指令翻译为机器码，计数器就是jvm内存模型对硬件寄存器的抽象实现
* 多线程执行jvm指令时，每个线程的计数器是独立的，互不影响
* 该区域没有任何OutOfMemoryerror

## 虚拟机栈
* 与数据结构中定义的栈有相同的含义，先进后出的结构，支持两种操作：出栈、入栈
* 描述方法调用的执行过程，每个方法在执行时，会将方法参数、局部变量、操作数栈、方法出口等信息封装为栈帧（Stack Frame）
* 方法的调用对应着栈帧的入栈和出栈
* 栈帧的核心是局部变量表，存在编译期可知的Java基本类型数据、对象引用、returnAddress类型，这些数据存储的单位成为slot，64位长度的long、double占两个slot，其余的占一个slot
* 局部变量表空间在编译期完成分配，运行期不会改变
* 超过调用栈最大深度抛出StackOverFlowError，无法申请到内存抛出OutOfMemoryerror

## 本地方法栈
* 作用与虚拟机栈一致，hotspot将本地方法栈和虚拟机栈合二为一，没有单独实现本地方法栈
* 超过调用栈最大深度抛出StackOverFlowError，无法申请到内存抛出OutOfMemoryerror

## 堆
* 该区域唯一的目的就是存放对象，在GC分代收集思想下，堆又被各个垃圾收集器划分为新生代、老年代、永久代...等多个区域
* 线程共享，但是为了提高对象分配效率，TLAB可以划分出线程私有的堆空间
* 无法申请到内存抛出OutOfMemoryerror

## 方法区
* 存储类加载信息、常量、静态变量等
* 在jdk1.8前，方法区的实现方式是用永久代，当时两者并不等价，只是由于在GC分代思想下，将方法区设置在永久代中方便垃圾收集器管理
* 从jdk1.8开始，完全废弃永久代概念，将类的加载信息移到元空间，将字符串常量池、静态变量移到堆空间
* 无法申请到内存抛出OutOfMemoryerror

### 运行时常量池
* 方法区的一部分，Class文件中有一个区域叫常量池，用于存放编译期生成的字面量和符号引用，这部分内容在类加载后存储到运行时常量池
* 运行时常量池具有动态扩展性，比如调用String.intern()将字符串加入到常量池，但池Class中的常量池编译期就可获得，没有扩展性
* 无法申请到内存抛出OutOfMemoryerror

## 直接内存
* 不是java虚拟机规范中定义的区域，使用本机内存，无法申请到内存抛出OutOfMemoryerror

# jvm连载：对象的创建
## new指令执行过程
1. 检查指令参数在常量池中是否有一个符号引用
2. 检查符号引用的类是否已被加载、链接、初始化
3. 如果没有加载，必须先执行加载过程
4. 为对象分配内存，对象所需的内存大小在类被加载完成后便可确定
5. 将对象中的字段都初始化为零值
6. 设置对象头信息：哪类的实例、对象的哈希吗、GC分代年龄、锁信息
7. 调用`Class<init>()`执行构造方法初始化

## 对象内存分配方式
### 指针碰撞
* 堆内存空间绝对规整，没有内存碎片
* 被使用过的空间放在左边，空闲的内存放在右边，中间使用指针作为分界点，分配内存时仅仅是把指针向右移动一段对象大小空间
* 带压缩整理的垃圾收集器使用这种方式
### 空闲列表
* 使用内存和空闲内存交错在一起
* jvm维护一个列表，记录哪块内存可用，在分配时找到一块足够大小的空间分给对象
* 没有压缩真理的垃圾收集器使用这种方式
### TLAB
* 在jvm中对象创建是频繁的操作，即使只修改指针的位置都有可能造成线程不安全
* 为每个线程预先在堆内存空间中分配一小块区域，分配空间时优先使用这块空间
* TLAB空间用完后，分配内存时采用cas+失败重试保证原子性操作

## 对象内存布局
* 对象头
    * mark word：哈希码、GC分代年龄、锁状态标志
    * 类型指针：类型元数据的指针，确定该对象是哪个类的实例
    * 长度：如果是数组类型还必须记录数组的长度
* 实例数据
    * 类型字段信息
* 对齐填充
    * 占位符，对象的大小必须是8的倍数

# jvm连载：垃圾收集器

# jvm连载：常用的监控命令
## jps jvm进程状态
* 与linux中的ps命令相似，显示正在运行的jvm进程
* 可选参数
    * `-q`：只显示进程id，不显示主类名称
    * `-m`：显示main()的启动参数
    * `-l`：显示主类的全名，如果是jar包，显示jar的路径
    * `-v`：显示jvm启动时的参数

## jstat jvm统计信息
* 显示jvm进程中的类加载、内存、垃圾收集、即时编译器等信息
* 可选参数
    * `-class`：类加载、卸载数量、占用空间大小及装载耗费时间
    * `-gc`：整个堆的gc信息
    * `-gcnew`：新生代gc信息
    * `-gcold`：老年代gc信息
    * `-gcutil`：整个堆使用空间占总空间的百分比
    * `-gccase`：与gcutil一致，增加导致上一次的gc的原因
    * `-gccapacity`：堆空间各个区域使用的最大、最小空间
    * `-gcnewcapacity`：新生代区域使用的最大、最小空间
    * `-gcoldcapacity`：老年代区域使用的最大、最小空间

## jinfo jvm配置信息
* 实时查看、调整jvm各项参数
* 可选参数
    * `-flags`：查看jvm启动参数配置
    * `-flag`：查看参数具体值
    * `-XX:+PrintFlagsInitial`：查看所有参数的默认值
    * `-XX:+PrintFlagsFinal`：查看所有参数的最终值

## jmap 内存映射信息
* 用于生成堆转储快照
* 可选参数
    * `-dump:[live],format=b,file=filename`：dump内存苦快照，live表示存活对象
    * `-heap`：显示堆详细信息
    * `-histo`：显示堆中对象统计信息
    * `-F`：强制执行

## jstack 线程快照信息
* 显示jvm中每一个线程正在执行的方法堆栈信息
* 可选参数
    * `-l`：锁的附加信息
    * `-F`：强制执行

# jvm连载：垃圾回收
## GC Root

# jvm连载：类加载器
* 数组类型本身不是通过类加载创建的，是由jvm直接在内存中动态构造出来的
* 一个类型必须与类加载器一起确定唯一性

# jvm连载：<clinit>()调用细节
* 由编译器自动收集类中的所有类变量的赋值动作和`static{}`中的语句合并而来，代码的顺序就是编译器执行的顺序
* `static{}`中只能访问到定义在静态块之前的变量，定义在静态块后的变量只能赋值，不能访问
```java
static int i = 0;
static{
    i =10;
    System.out.println(j); // 编译器报错Illegal forward reference
    j = 10; // 可以赋值
}
static int j;
```
* `<clinit>()`不需要显示的调用父类的`<clinit>()`，jvm保证子类的`<clinit>()`执行前，父类的`<clinit>()`先被执行，所以jvm中第一个被调用的`<clinit>()`是Object
* 父类的`<clinit>()`先被执行，就意味着静态块优先于子类的赋值动作先发生
```java
class SuperClass {
    public static int value = 10;

    static {
        value = 2;
    }
}

class SubClass extends SuperClass {
    public static int result = value;
}
```
* 如果类中没有`static{}`，也没有变量的赋值，编译器可以不生成`<clinit>()`
* 接口中不能定义`static{}`但仍然有初始化赋值的操作，一样会生成`<clinit>()`，接口的实现类在初始化时不会执行接口的`<clinit>()`
* jvm保证在一个类的`<clinit>()`被多线程执行时被正确的加锁同步，同一个类加载器下，一个类的`<clinit>()`只会被执行一次，如果在`<clinit>()`中有耗时很长的操作，在初始化阶段可能造成线程阻塞




# jvm连载：引用
## 引用定义
* 如果reference类型的数据中存储的值代表另外一块内存的起始地址，该reference数据就代表某个对象的引用

## 引用的局限
* reference只定义了被引用、未引用两种状态
* 当内存空间足够时，保留在内存中，当GC后内存空间紧张，回收掉一部分引用对象，reference的定义就显得有些局限
* java对reference的被引用、未引用进行扩展，定义出强引用、弱引用、软引用、虚引用

## 强引用
* 这种引用关系就是reference定义的引用，类似`Object obj = new Object()`
* 被GC Root关联的对象都属于强引用，无论什么情况下只要强引用关系存在，对象都不会被回收

```java
// -Xmx20M -XX:+PrintGCDetails -verbose:gc
private static final int _4MB_LENGTH = 1024 * 1024 * 4;
List<byte[]> list = new ArrayList<>();
for (int i = 0; i <= 5; i++) {
    list.add(new byte[_4MB_LENGTH]);
    System.out.println("loop i=" + i);
}
```
* 程序在循环到第3次时直接OOM，20m的内存空间无法分配6个4m的对象

## 软引用
* 描述一些在内存充足的情况下存活，在内存紧张时回收的对象，在GC后内存空间仍然不足时，会再次触发GC回收软引用的对象
* SoftReference实现软饮用

```java
// -Xmx20M -XX:+PrintGCDetails -verbose:gc
private static final int _4MB_LENGTH = 1024 * 1024 * 4;
List<SoftReference<byte[]>> list = new ArrayList<>();
for (int i = 0; i <= 5; i++) {
    list.add( new SoftReference<>(new byte[_4MB_LENGTH]));
    for (SoftReference ref : list) {
        System.out.print(ref.get() + "|");
    }
    System.out.println();
}
System.out.println("list.size=" + list.size());
for (SoftReference<byte[]> ref : list) {
    System.out.println("ref=" + ref + ",ref.get()=" + ref.get() + " | ");
}
```
* 从以下的GC日志可以看出，带循环低4次后发生了GC，回收到前4次分配的16M数组
* 6次循环后，SoftReference中前4次的byte[]已经别回收

## 弱引用
* 比软引用强度还低，在GC时无论内存是否够用，软饮用对象都会被回收
* WeakReference实现软引用

```java
// -Xmx20M -XX:+PrintGCDetails -verbose:gc
private static final int _4MB_LENGTH = 1024 * 1024 * 4;
List<WeakReference<byte[]>> list = new ArrayList<>();
for (int i = 0; i <= 5; i++) {
    list.add(new WeakReference<>(new byte[_4MB_LENGTH]));
    for (WeakReference ref : list) {
        System.out.print(ref.get() + "|");
    }
    System.out.println();
}
System.out.println("list.size=" + list.size());
for (WeakReference<byte[]> ref : list) {
    System.out.println("ref=" + ref + ",ref.get()=" + ref.get() + " | ");
}
```
* 第1次循环添加4m对象后发生GC，4m对象马上被回收
* 第4次循环后由于内存空间不足以容纳4m对象，又触发GC，回收到第2次循环添加到集合的对象

## 虚引用
* 最弱引用关系，无法通过虚引用获得一个对象实例，必须配合引用队列使用
* PhantomReference实现虚引用

## 终结器引用
* Object类里都有finallize()方法，对象重写了finallize()并且没有强引用后，进行垃圾回收时，由虚拟机创建一个终结器引用，当这个对象被垃圾回收时，会把终结器引用加入到引用队列，不需要编码，必须关联引用队列使用
* 在第一次GC时，对象的终结器引用加入引用队列，由优先级很低的Finalizer线程通过终结器引用调用finallize()，第二次GC时才能回收该对象
## 引用队列
* 当引用的对象将要被jvm回收时，会将其加入到引用队列中，对要jvm被回收的对象提供额外的操作
* 在软引用的例子中，List中的SoftReference持有的byte[]已经被回收，但是SoftReference仍然得不到回收，通过引用队列回收SoftReference

```java
List<SoftReference<byte[]>> list = new ArrayList<>();
ReferenceQueue<byte[]> queue = new ReferenceQueue();
for (int i = 0; i <= 5; i++) {
    list.add(new SoftReference<>(new byte[_4MB_LENGTH], queue));
    for (SoftReference ref : list) {
        System.out.print(ref.get() + "|");
    }
    System.out.println();
}
System.out.println("list.size=" + list.size());
Reference<byte[]> temp = (Reference<byte[]>) queue.poll();
while (temp != null) {
    list.remove(temp);
    temp = (Reference<byte[]>) queue.poll();
}
for (SoftReference<byte[]> ref : list) {
    System.out.println("ref=" + ref + ",ref.get()=" + ref.get() + " | ");
}
```
* 由于byte[]被回收，SoftReference在list集合中也会被回收

# jvm连载：类主动初始化时机
* 类的加载过程分为`加载`、`链接`、`初始化`，链接又细分为`验证`、`准备`、`解析`三个阶段
* `《java虚拟机规范》`中规定了6种情况，必须对类进行初始化
## 第一种情况
* 遇到`new`、`getstatic`、`putstatic`、`invokestatic`四条指令时
    * new关键词实例化对象
    * 读取设置静态类型字段（编译器把结果放入常量池的静态字段除外）
    * 调用一个静态方法
## 第二种情况
* 使用java.lang.reflect包的方法对类进行反射操作时
## 第三种情况
* 初始化一个类时，如果父类未初始化，优先初始化父类
## 第四种情况
* jvm启动时的主类，包含main()
## 第五种情况
* 对于动态语言的支持
## 第六种情况 
* 当一个接口定义了default方法，接口实现类初始化时，接口要先被初始化

## 被动初始化例子一
```java
class SuperClass{
    public static int value =10;
    static{
        System.out.println("super class");
    }
}
class SubClass extends SuperClass{
    static {
        System.out.println("sub class");
    }
}
```
* 当调用`SubClass.value`时，只会导致SuperClass初始化，不会导致SupClass初始化
* 对于static字段，只有直接定义的类才会初始化

## 被动初始化例子二
* 调用`SuperClass[] arr = new SuperClass[2]`时，不会导致SuperClass初始化
* 初始化某类的数组，不会导致该类被初始化

## 被动初始化例子三
```java
class ConstClass{
    public static final int value =10;
    static{
        System.out.println("super class");
    }
}
public class NotInit {
    public static void main(String[] args) {
        System.out.println(ConstClass.value);
    }
}
```
* 在NotInit类中调用`ConstClass.value`不会导致SuperCass初始化
* 在编译期将`ConstClass`中的常量value直接保存到了`NotInit`类的常量池中
* 编译期过后`ConstClass`与`NotInit`将毫无关系


# jvm连载：字符串常量池与String对象
```java
String s1 = "a";
String s2 = "b";
String s3 = "ab";
String s4 = s1 + s2; // 变量在运行时可能被修改
System.out.println(s3==s4);
```
## 
```java
String s1 = "a";
String s2 = "b";
String s3 = "ab";
String s4 = "a" + "b"; // 编译期间结果确定（编译器优化）
System.out.println(s3==s4);
```  
## 
```java
final String s1 = "a";
final String s2 = "b";
String s3 = "ab";
String s4 = s1 + s2; // s1、s2是常量时，编译期结果可以确定
System.out.println(s3==s4); 
```    
## 
```java
String s1 = new String("a")+new String("b");
s1.intern();
String s2="ab";
System.out.println(s1==s2);
System.out.println(s1=="ab");
```
在程序一开始声明String s="ab"，intern()返回的是字符串常量池的对象
1.8 intern()
    string table存在，不放入
    string table不存在，放入string table，返回string table中的对象
1.6 inter()
    string table不存在，复制对象放入string table，返回string table中的对象 
    String s = "a";
    s==s.intern(); // false

## 延迟初始化
```java
System.out.println("1");
System.out.println("2");
System.out.println("3");
System.out.println("4");
System.out.println("5");
System.out.println("6");
System.out.println("7");
System.out.println("8");
System.out.println("9");
System.out.println("10");

System.out.println("1");
System.out.println("2");
System.out.println("3");
System.out.println("4");
System.out.println("5");
System.out.println("6");
System.out.println("7");
System.out.println("8");
System.out.println("9");
System.out.println("10");
```

## 常量池压栈操作
栈的宽度为4字节
bipush byte
sipush short
ldc int
ldc2_w long 两次压栈，long长度是8字节
超过short范围的值存在常量池，小于short范围的值存在.class文件中（运行后都应该存在于常量池中）

## 局部变量表入/出栈操作
iload int 将局部变量中的值保存到操作数栈
istore int 弹出操作数栈顶元素，保存到局部变量表x slot位置


## 其它
iadd 加法运算

int a = 10;
int b = a++ + ++a + a--


对象头：8/16
mark word：4/8
klass：4/8
length：4
8的倍数