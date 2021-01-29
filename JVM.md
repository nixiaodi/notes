# JVM

## 1 类加载器及加载过程

### 1.1 类加载子系统作用

- 负责从文件系统或网络中加载class文件，class文件开头有特定的标识

- classLoader只负责class文件的加载，至于它是否可以运行，由执行引擎决定

- 加载的类信息存到内存--方法区，除了类信息，方法区还会存放运行时常量池信息，还可能包括字符串字面量和数字常量

  `常量池运行时加载到内存中，即运行时常量池`

- 类加载器加载字节码文件到内存

### 1.2 类加载器角色

![image-20210126160354230](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126160354230.png)

> 理解下来就是类加载器加载class文件，在方法区中生成了一个Class对象以及方法信息等，然后通过Class对象可以获取到对应的类加载器，也可以通过实例化在堆中生成对应的对象

### 1.3 类加载的执行步骤

- 加载

  通过一个类的全限定名获取定义此类的二进制文件

  将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

  `在内存中生成一个代表这个类的Class对象，作为方法区这个类的各种数据访问入口`

- 链接

  - 验证

    确保Class文件的字节流中包含信息符合虚拟机要求，保证加载类的正确性

    主要包括四种验证：文件格式验证、元数据验证、字节码验证、符号引用验证

  - 准备

    为类中静态变量分配内存，并设置静态变量初始值，即零值

    `不包括final修饰的static，因为final在编译的时候就分配内存，准备阶段会显式初始化`

    `不会为实例变量分配初始化`，类变量会分配在方法区中，而实例变量会随着对象一起分配到堆中

  - 解析

    虚拟机将常量池中的`符号引用替换为直接引用`的过程(符号引用理解为一个标识，而在直接引用直接指向内存中的地址)

    解析一般会随着JVM的初始化结束之后再执行

    解析动作主要针对类或接口、字段、类方法、接口方法和方法类型等

- 初始化

  初始化阶段就是执行类构造器方法`<clinit>`的过程

  `<clinit>`方法不需要定义，是javac编译器自动收集类中的所有`静态变量赋值和静态代码块中的语句`合并而来

  也就是说如果一个类中没有静态变量以及静态代码块，也就不存在`<clinit>`方法

  JVM必须保证一个类的`<clinit>`方法在多线程下被同步加锁

#### 1.3.1 初始化测试

```java
public class Test01 {
    private static int num = 1;

    static {
        num = 2;
    }

    public static void main(String[] args) {
        System.out.println(Test01.num);
    }
}
```

使用jclasslib查看编译之后的结果：

![image-20210126164153228](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126164153228.png)

> 说明：一个类中可能没有clinit方法，但是一定存在构造器(init)方法，即使自身没有构造器，也会使用默认的无参构造器

### 1.4 类加载器分类

通过类的全限定名获取该类的二进制字节流的代码块叫做类加载器

![image-20210126164547028](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126164547028.png)

- 引导类加载器：用来加载java核心类库，无法被java程序直接引用，底层使用C/C++编写
- 扩展类加载器：用来加载java的扩展库，JVM的实现会提供一个扩展库目录，该类加载器在此目录里面查找并加载java类
- 系统类加载器：根据java应用的类路径(CLASSPATH)加载java类，一般java应用都是通过它来进行加载
- 用户自定义类加载器：通过集成java.lang.ClassLoader类的方法实现

#### 1.4.1 测试类加载器

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        ClassLoader classLoader = ClassLoader.getSystemClassLoader();
        System.out.println(classLoader);

        ClassLoader loader = ClassLoaderTest.class.getClassLoader();
        System.out.println(loader);

        ClassLoader parent = loader.getParent();
        System.out.println(parent);

        ClassLoader parent1 = parent.getParent();
        System.out.println(parent1);
    }
}
```

结果：

```
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@2ff4acd0
null
```

> 获取系统类加载器就是AppClassLoader
>
> 而我们自定义的类的加载器也是AppClassLoader，AppClassLoader的父加载器是ExtClassLoader
>
> ExtClassLoader的父加载器应该是引导类加载器，但是由于引导类加载器是通过C/C++编写，因此获取为null

> 也可以看出获取类加载器的方式：
>
> `ClassLoader.getSystemClassLoader()`通过ClassLoader的静态方法获取系统类加载器
>
> `ClassLoaderTest.class.getClassLoader()`通过对象的Class对象获取
>
> `getParent()`通过一个类加载器获取其父类加载器
>
> `new ClassLoaderTest().getClass().getClassLoader()`

```java
ClassLoader loader = Thread.currentThread().getContextClassLoader();
```

也可以通过当前线程获取线程上下文的classLoader

### 1.5 双亲委派机制

JVM对class文件采用的是`按需加载`的方式，当需要使用该类时才会将它的class文件加载到内存生成Class对象；而加载某个类的class文件时使用的就是双亲委派模式

假设一种场景，我们自定义一个String类，且包名也是java.lang

```java
package java.lang;
public class String {
    static {
        System.out.println("我是自定义的String类");
    }
}
```

再另外一个类中创建一个String对象进行测试：

```java
public class StringTest {
    public static void main(String[] args) {
        String str = new String();
        System.out.println("hello");
    }
}
```

> 结果中并没有执行自定义String类中的静态代码块，说明并没有使用我们自定义的String类

那么之所以没有使用自定义的String类就是由于双亲委派机制

#### 1.5.1 双亲委派机制工作原理

![u=2699065805,3218366095&fm=15&gp=0](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/u=2699065805,3218366095&fm=15&gp=0.jpg)

- 如果一个类加载器收到类加载请求，它不会自己先去加载，它会先把这个请求传递给它的父类加载器去执行。
- 如果父类加载器还存在父类加载器，则继续向上委托，依次传递，请求最终到达顶层的启动类加载器
- 如果父类加载器可以完成类加载的任务，就成功返回
- 倘若父类加载器无法完成这个加载任务，子加载器就会去尝试加载，如果还不行，就子子加载器尝试加载，直到成功加载

#### 1.5.2 优势

- 避免类的重复加载
- 保护程序安全，防止核心API被随意篡改(如java.lang.String)

#### 1.5.3 沙箱安全机制

上次的例子自定义的String类再添加main方法执行

```java
public class String {
    static {
        System.out.println("我是自定义的String类");
    }

    public static void main(String[] args) {
        System.out.println("test");
    }
}
```

结果：

```
错误: 在类 java.lang.String 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

> 说明引导类加载器发现该类在java.lang包下就准备加载，然而并不会加载自定义的String类，在原有的String类中并没有main方法就会报错

### 1.6 补充

在JVM中表示两个class对象是否为同一个类存在两个必要条件：

- 类的完整类名必须一致，包括包名
- 加载这个类的ClassLoader实例对象必须相同

类的使用方式：

主动使用：

- 创建类的实例
- 访问类或接口的静态变量，或对该静态变量赋值
- 调用类的静态方法
- 反射(如Class.forName())
- 初始化一个类的子类(调用父类的构造器)
- Java虚拟机启动时被表明为启动类的类
- 动态语言支持

被动使用：

​		除去主动使用就是被动使用

`主动使用和被动使用的区别：被动使用不会导致类的初始化`

## 2 PC寄存器

### 2.1 扩展

#### 2.1.1 运行时数据区概述

![u=1760198352,2914405773&fm=11&gp=0](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/u=1760198352,2914405773&fm=11&gp=0.jpg)

- 程序计数器：记录下一条字节码执行指令，实现分支循环跳转、异常处理、线程恢复等功能
- 虚拟机栈：存储局部变量表、操作数栈、动态链接、方法返回地址等信息
- 本地方法栈：本地方法调用
- 堆：所有线程共享，几乎所有对象实例都在堆中分配内存
- 方法区：用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译后的代码等数据

##### 2.1.1.1 不同区和线程之间的关系

- 线程私有：程序计数器、虚拟机栈、本地方法栈
- 线程共享：堆、方法区

##### 2.1.1.2 堆和栈的区别

- 物理地址

  堆：对象分配物理地址不连续，性能相对栈较弱

  栈：先进后出，物理地址连续，性能相对堆较好

- 内存分配

  堆：在运行时分配，大小不固定

  栈：在编译时分配，大小固定

- 存放内容：

  堆：对象的实例和数组，更关注数据存储

  栈：局部变量、操作数、动态链接、方法返回地址等信息

- 程序可见性

  堆：所有线程共享，可见

  栈：线程私有，只对线程可见，生命周期和线程相同

##### 2.1.1.3 深拷贝和浅拷贝

- 浅拷贝：增加一个指针指向已有的内存地址
- 深拷贝：增加一个指针指向新开辟的一块内存空间

原内存发生变化，浅拷贝随之变化；深拷贝则不会随之发生变化

#### 2.1.2 JVM中的线程

JVM中的每个线程都和操作系统的本地线程直接映射(`类似于用户线程和内核线程的映射，一对一`)

一旦本地线程初始化成功(分配完资源，创建了TCB)，它就会调用Java线程中的run()方法

### 2.2 PC寄存机概述

`寄存器`用于存储指令相关的现场信息，CPU只有把数据装载到寄存器才能运行

JVM的PC寄存器是对物理PC寄存器的一种抽象模拟

PC寄存器用来存储指向下一条指令的地址，也就是将要执行的代码，由操作引擎读取指令

![image-20210127144926462](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127144926462.png)

特点：

- 运行时数据区最小的一块内存区域，几乎可以忽略不计，也是运行速度最快的内存区域
- 每个线程有一个私有的程序计数器，线程之间不互相影响
- 运行时数据区唯一不会出现OOM的区域，没有垃圾回收
- 程序计数器会存储当前线程正在执行的Java方法的JVM指令地址
- 如果正在执行本地方法，则计数器的值应为空(undefined)
- 字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令

### 2.3 测试

```java
public class PcRegisterDemo {
    public static void main(String[] args) {
        int i = 10;
        int j = 20;
        int k = i + j;
        
        String s = "hello";

        System.out.println(i);
        System.out.println(k);
    }
}
```

对这个类的class文件进行反编译：`javap -v PcRegisterDemo.class`

结果：

![image-20210127150749394](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127150749394.png)

> 首先方法中记录着每条指令的指令地址和操作指令，而PC寄存器记录着下一条操作指令的指令地址
>
> 执行引擎就会根据PC寄存器的指令地址去虚拟机栈中操作局部变量表等
>
> 最后转化为机器指令，让CPU去执行机器指令

### 2.4 相关问题

PC寄存器存储字节码指令的作用是什么？

- 线程是一个个的顺序执行流，CPU需要不停的切换各个线程，切换回来的时候就知道接着从哪开始执行
- JVM的字节码解释器需要通过`改变PC寄存器的值`来明确下一条应该执行什么样的字节码指令
- 记录下一条字节码执行的指令，实现分支循环跳转、异常处理、线程恢复等功能

PC寄存器为什么设定为线程私有？

- CPU为每个线程分配时间片，多线程在特定的时间段内只会执行某一个线程的方法，CPU会不停地进行任务切换，线程需要中断、恢复(`中断指令，外中断`)

- 各个线程、PC寄存器记录的当前执行指令地址可以独立进行计算，防止互相干扰

## 3 虚拟机栈

栈解决程序的运行问题，即程序如何执行，或者如何处理数据

每个线程在创建时都会创建一个虚拟机栈，`内部保存着一个个的栈帧`

线程私有，生命周期和线程一致

主管Java程序的运行，保存方法的局部变量(`8种基本数据类型、对象的引用地址`)，部分结果，并参与方法的调用和返回

### 3.1 栈的特点

- 快速有效的存储方式，访问速度仅次于程序计数器

- JVM直接对Java栈的操作只有两个

  - 每个方法执行，伴随着进栈
  - 执行结束之后的出栈

- 栈不存在垃圾回收，但是存在OOM、栈溢出

  ![image-20210127154827202](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127154827202.png)

  > 栈的大小是动态或者固定不变的
  >
  > - 如果是动态扩展，可能无法申请到足够内存出现OOM
  > - 如果是固定，可能线程的请求栈容量超过固定值，出现StackOverflowErroe

### 3.2 测试栈溢出

**在启动配置中配置JVM的启动参数修改栈内存的大小**：`-Xss MaxStackSize(如256K)`

![image-20210127155834218](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127155834218.png)

测试程序为：

```java
public class StackErrorDemo {
    private static int count = 0;

    public static void main(String[] args) {
        System.out.println(count);
        count++;
        main(args);
    }
}
```

> 这里主要是通过不断的递归调用main函数来测试栈的深度

当设置栈内存为256k时：

![image-20210127160018202](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127160018202.png)

> 栈的深度为2457时便出现了栈溢出

当设置栈内存为1m时：

![image-20210127160116267](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127160116267.png)

> 栈的深度为11413时才出现栈溢出

### 3.3 栈的存储结构

每个线程都有自己的栈，栈中的数据是以栈帧的格式进行存储

栈帧：

- 在这个线程上执行的每个方法都对应着一个栈帧
- 栈帧是一个内存区快，是一个数据集，维持着方法执行过程中的各种数据信息
- JVM对虚拟机栈的操作只有压栈和弹栈，且遵循"先入先出"
- 一条活动的线程中，一个时间点上，只有一个活动的栈帧；只有当前正在执行的方法的栈顶栈帧是有效的，也就是`当前栈`，对应的方法是`当前方法`，对应的类是`当前类`
- 执行引擎运行的所有`字节码指令只针对当前栈帧`进行操作(`同样，寄存器中存储的也是当前栈的指令地址`)
- 如果方法中调用了其他方法，对应的新栈帧就会被创建出来，放在顶端成为新的当前栈

### 3.4 栈运行原理

当前方法调用了其他方法，方法返回之际，当前栈帧会回传方法的执行结果给前一个栈帧，接着虚拟机会丢弃当前栈帧，使得前一个栈帧重新成为新的当前栈

`不同线程包含的栈帧不允许相互引用`

Java方法有两种返回方式：

- 一种是正常的函数返回，使用return指令

- 一种是抛出异常(**没有处理的exception，error无法被处理**)，不管哪种方式，都会导致栈帧被弹出

  > 从而可以理解当前栈出现异常且没有处理时，当前栈就会被弹出，异常就会交给前一个栈帧...一直这样进行下去直到main方法，如果main方法也没有处理，就停止JVM
  >
  > 这里的处理就是指的try...catch...

### 3.5 栈帧的内部结构

- 局部变量表
- 操作数栈
- 动态链接(指向运行时常量池的方法引用)
- 方法返回地址(方法正常退出或异常退出的定义)
- 一些附加信息

#### 3.5.1 局部变量表

定义为一个`数字数组`，主要用于存储方法参数，定义在方法体内部的局部变量，数据类型包括各类基本数据类型，对象引用以及return address类型

局部变量表建立在线程的栈上，是线程私有的，因此`不存在数据安全问题`(因为没有线程之间数据共享)

局部变量表容量大小是在编译期确定下来的，并保存在方法的Code属性的maximum local variables数据项中，方法运行期间不会改变

##### 3.5.1.1 测试局部变量表的容量

```java
public class LocalVariblesDemo {
    public static void main(String[] args) {
        LocalVariblesDemo demo = new LocalVariblesDemo();
        int num = 10;
        demo.test01();
    }

    public void test01() {
        String str = "hello";
        System.out.println(str);
    }
}
```

通过jclasslib反编译之后观察结果：

![image-20210127172157509](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127172157509.png)

![image-20210127172221709](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210127172221709.png)

> 这里可以看出有局部变量表的容量，以及局部变量表中存储的数据信息，包括方法的参数，以及创建的局部变量

`栈帧的大小主要受局部变量表大小的影响`，从而当栈内存大小确定之后，每个栈帧的大小决定了整个栈中栈帧的数量

当方法调用结束之后，随着方法栈帧的销毁，局部变量表也会随之销毁

##### 3.5.1.2 字节码方法内部结构解析

![image-20210128141151203](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128141151203.png)

> 当点击方法时可以看到
>
> 1、name是定义该方法的名字
>
> 2、描述符是定义该方法的参数，`[`代表数组，`L`代表引用类型，从而说明参数是一个String数组
>
> 3、访问标志也就是访问控制符的关键字

![image-20210128141728932](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128141728932.png)

> 字节码也就是反编译的字节码指令地址和指令
>
> 异常表就是该方法出现的异常信息
>
> 杂项里面包括操作数栈大小、局部变量表大小以及字节码长度

![image-20210128142023187](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128142023187.png)

> Java代码的行号和字节码指令行号的对应关系

![image-20210128142220357](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128142220357.png)

> 这里主要存储局部变量表的信息
>
> name代表局部变量的名字，序号代表局部变量在局部变量表数组中的索引位置
>
> 起始PC代表的是局部变量在字节码指令中的行号，而长度代表的是局部变量从创建开始之后作用作用域范围(`可以看出如果在方法中定义的变量如果没有加代码块，起始PC+长度刚好等于字节码的长度`)

##### 3.5.1.3 变量槽Slot

局部变量表最基本的存储单位就是Slot(变量槽，本质上也就是数组中的一个个位置)

局部变量表中存放编译期可知的各种基本数据类型(8种)，引用类型(reference)，returnAddress类型的变量

在局部变量表中，32位以内的类型只占用一个Slot(包括returnAddress类型和引用类型)，64位类型(long和double)占用两个Slot、

- byte、short、char在存储前会被转化为int，boolean也会被转为int

> 怎么去理解32位以内的占一个Slot呢？
>
> 我们知道局部变量表是一个int数组，int类型占用4个字节(也就是32bit，也就是32位)
>
> 而long和double占用8个字节，从而必须要占用两个Slot

JVM会为局部变量表中的每一个Slot分配一个索引，通过这个索引即可成功访问到局部变量表中指定的局部变量

当一个实例方法被调用时，它的方法参数和方法体内定义的局部变量会`按照顺序`被复制到局部变量表中的每个Slot上

![image-20210128143755394](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128143755394.png)

> 这里需要注意的是如果存储的是long或者double类型，只需要使用前一个索引
>
> `如果当前帧是由构造方法或实例方法创建，那么该对象引用this会存放在index为0的Slot处，其余的参数按照参数表顺序进行存储`

测试构造方法或实例方法的Slot：

![image-20210128145154475](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128145154475.png)

> init构造器方法和test01实例方法局部变量表索引为0都是变量this，从而也说明了为什么静态方法不能使用this变量

Slot的重复利用：

`局部变量表中的槽位是可以重用的`，如果一个局部变量过了其作用域，那么在其作用域之后申明的新局部变量就会可能复用过期局部变量的槽位，从而节省资源

```java
public void test02() {
    {
        int a = 1;
        System.out.println(a);
    }
    int b = 3;
}
```

对于test02方法的局部变量表应该是什么样的呢？

![image-20210128145758339](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128145758339.png)

> 索引为1的Slot先是被变量a使用，之后又被变量b使用

##### 3.5.1.4 静态变量--局部变量

- 静态变量不需要在使用前就进行显式赋值，而局部变量在使用前必须要进行显式赋值

  > 原因就是静态变量在链接--准备阶段会进行默认赋值
  >
  > 而局部变量不会进行默认赋值，所以使用前必须要进行赋值！

  ![image-20210128150409776](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128150409776.png)

  

##### 3.5.1.5 补充

栈帧中与`性能调优关系最密切的就是局部变量表`，在方法执行时虚拟机使用局部变量表完成方法的传递

`局部变量表中的变量也是重要的垃圾回收根节点，只要被局部变量表直接或间接引用的对象都不会被回收`

#### 3.5.2 操作数栈

操作数栈是通过数组实现的(`逻辑结构是栈，存储结构是数组`)，因此操作数栈既有栈的特点(先进后出)，也有数组的特点(按照顺序存放，带有索引)

操作数栈在方法执行过程中根据字节码指令，往栈中写入数据或提取数据，即入栈和出栈

- 某些字节码指令将值压入操作数栈，其余的字节码指令将操作数取出栈，使用它们后再把结果压入栈
- 比如：指定复制、交换、求和等操作

操作数栈主要是用于`保存计算过程的中间结果`，同时作为`计算过程中变量临时的存储空间`

操作数栈就是JVM执行引擎的一个工作区，当一个方法刚开始执行的时候，一个新的栈帧也被创建出来，`这个方法的操作数栈是空的(空数组)`

每一个操作数栈都会`拥有一个明确的栈深度`用于存储数值，其`所需的最大深度在编译期就确定好了`，保存在方法的Code属性中，为max_stack的值

![image-20210128161251411](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128161251411.png)

> 编译期已确定，且运行时不会变化

栈中的任何一个元素都可以是Java的任意数据类型(和局部变量表的规则一致)

- 32bit的类型占用一个栈单位深度
- 64bit的类型占用两个栈单位深度

`操作数栈并非采用访问索引的方式来进行数据访问`，而只能通过标准的入栈和出栈进行操作访问(`从而说明操作数栈虽然是用数组实现的，然而却需要使用栈的特性`)

`如果被调用的方法带有返回值的话，其返回值也会被压入当前栈帧的操作数栈中`，并更新PC寄存器中下一条需要执行的字节码指令

Java的虚拟机解释引擎是基于栈的执行引擎，其中的栈就是指操作数栈

#### 3.5.2.1 测试操作数栈

```java
public class OperandStackDemo {
    public static void main(String[] args) {
        byte i = 15;
        int j = 20;
        int k = i + j;
    }
}
```

先来分析下main方法的字节码指令：

![image-20210128154142628](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128154142628.png)

> 从指令地址2、5中可以看出中可以看出byte、short、char、int最后都是以int来进行保存

指令步骤解析：

![1611820749(1)](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/1611820749(1).jpg)

> 对于指令地址0和2解析：
>
> 指令地址为0时，将15放入操作数栈中，此时局部变量表为空
>
> 指令地址为2时，将15从栈顶弹出放入局部变量表中，操作数栈为空
>
> `之所以15放在局部变量表索引为1的位置，是因为这是一个实例方法，索引为0的位置是this`

![image-20210128160439233](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128160439233.png)

> 对于指令地址3和5解析：
>
> 指令地址为3时，将8压入栈顶
>
> 指令地址为5时，将8弹出栈顶存储在局部变量表中，操作数栈为空

![image-20210128160656200](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128160656200.png)

> 对于指令地址6和7解析：
>
> 分别将局部变量表中索引为1和索引为2的局部变量压入操作数栈中

![image-20210128160848957](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128160848957.png)

> 对于指令地址8和9解析：
>
> 指令地址为8时，执行引擎操作操作数栈，对两个数据进行求和，将最新的23压入栈顶
>
> 指令地址为9时，操作数栈弹出23存储到局部变量表中

##### 3.5.2.2 i++和++i

- i++：现将i的值加载到操作数栈，再将i的值加1
- ++i：先将i的值加1，再将i的值加载到操作数栈

##### 3.5.2.3 栈顶缓存技术

基于栈式架构的虚拟机所使用的的零地址指令更加紧凑，但完成一项操作的时候需要使用更多的入栈和出栈指令，也就意味着需要更多的指令分派次数和内存读写次数

栈顶内存技术`将栈顶元素全部缓存在物理CPU的寄存器中`，依次降低对内存的读写次数，提升执行引擎的执行效率

#### 3.5.3 动态链接

每一个栈帧内部都包含一个`指向运行时常量池`的该栈帧所属方法的引用，包含这个引用的目的就是为了支持当前方法的代码能够实现动态链接

在Java源文件被编译为字节码文件时，所有的变量和方法引用都作为符号引用保存在class文件的常量池中(`比如一个方法调用了另外的其他方法时，就是通过常量池执行方法的符号引用来表示`)，动态链接的作用就是为了`将这些符号引用转换为调用方法的直接引用`

> 注意是运行时常量池，不是class文件的常量池

##### 3.5.3.1 测试动态链接和常量池的作用

```java
public class DynamicLinkingTest {
    int num = 10;

    public void methodA() {
        System.out.println("hello");
    }

    public void methodB() {
        System.out.println("yes");
        methodA();
        num++;
    }
}
```

对于methodB方法：

![image-20210128164241899](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128164241899.png)

> 这里有个`invokevirtual`对应着`#7`，从注释中可以看出这是在调用methodA
>
> 那么`#7`到底是什么呢？

![image-20210128164424578](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128164424578.png)

> 在编译之后可以看出这里存在`Constant pool`，也就是常量池，其中`Methodref`代表方法引用，紧接着又指向#8调用#31，而#8又指向#32

![image-20210128164654015](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128164654015.png)

> 从这里常量池的引用可以看出：
>
> 也就是`org/jiang/chapter01/DynamicLinkingTest`类中的`methodA`方法，参数值为()，返回值为void

##### 3.5.3.2 常量池理解

![image-20210128170805868](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128170805868.png)

为什么要使用常量池呢？

> 运行时常量池中存储一份，可以供多个虚拟机栈进行调用，节省资源
>
> 实际上运行时常量池中存放的也是引用，真正的变量存储在heap中
>
> 常量池提供符号、常量便于指令识别(`如果把所有的类信息、方法信息在调用时直接加载到内存中，占用的空间就会特别大`)

常量池、运行时常量池和字符串常量池：

[常量池](https://www.jianshu.com/p/8966c51e9728)

- 常量池

  即class文件常量池，是class文件的一部分，用于保存编译时确定的数据

  除了包含**类的版本、字段、方法、接口**等描述信息外，还有一项信息就是常量池(constant pool table)，用于存放编译器生成的各种**字面量**(文本字符串、被声明为final的常量、基本数据类型的值)和**符号引用**(类和接口的全限定名、字段的名称和描述符、方法的名称和描述符)

  ![image-20210128171841482](https://learning-1304029482.cos.ap-chengdu.myqcloud.com/learning/image-20210128171841482.png)

- 运行时常量池

  当类加载到内存中后，JVM就会将class常量池中的内容存放到运行时常量池中

  Java语言不要求常量一定只能在编译期产生，运行期也可能产生新的常量，这些常量被放在运行时常量池中

  class常量池中存的是字面量和符号引用，也就是说他们存的并不是对象的实例，而是对象的符号引用值。而经过解析（resolve）之后，也就是把符号引用替换为直接引用，解析的过程会去查询字符串常量池，也就是我们上面所说的string pool，以保证运行时常量池所引用的字符串与字符串常量池中所引用的是一致的

- 字符串常量池

  字符串常量池里的内容是在类加载完成，经过验证，准备阶段之后在堆中生成字符串对象实例，然后将该字符串对象实例的引用值存到String pool中

  String pool中存的是引用值而不是具体的实例对象，具体的实例对象是在堆中开辟的一块空间存放

  String pool在每个HotSpot VM的实例只有一份，被所有的类共享

