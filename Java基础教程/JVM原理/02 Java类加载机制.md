# Java类加载机制
> https://blog.csdn.net/qq_29167297/article/details/124800850
## 0 JVM简介
### JVM空间
JVM内存共分为虚拟机栈、堆、方法区、程序计数器、本地方法栈五个部分。

![](image/2022-12-04-13-14-56.png)

### ClassLoader

![](image/2022-12-04-13-15-46.png)
## 1 加载
加载阶段是将class文件从磁盘或者jar等读到JVM内存中，并为其创建一个Class对象。任何一个类被使用时候系统都会为其创建一个Class对象的。加载的同时将加载的这些数据转换成方法区中运行时数据(运行时候数据区：静态变量、静态代码块、常量池等)，作为方法区数据的访问入口。

加载是类加载的第一个过程，在这个阶段，将完成以下三件事情：

1. 通过一个类的全限定名获取该类的二进制流。
2. 将该二进制流中的静态存储结构转化为方法去运行时数据结构。
3. 在内存中生成该类的Class对象，作为该类的数据访问入口。


事实上，这三条限定都不是很严格，比如第一条，并没有明确指出通过全限定名从哪里得到二进制流，由此就有很多不同的实现：
* 在ZIP包中读取（JAR,EAR,WAR）
* 从网络中获取（APPLET）
* 运行时计算生成，这种场景使用的最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass来为特定接口申城$Proxy的代理类的二进制流
* 由其它文件生成（jsp）
* 从数据库中读取，有些中间件服务器（SAP NETWEAVER)

加载阶段完成后，虚拟机外部的二进制流就按照虚拟机所需的格式存储在方法区中，方法区中的数据存储格式由虚拟机实现自行定义。然后在JAVA堆中实例化一个java.lang.Class类对象（比如我们new A()对象，在加载过程中，会在堆区生成一个代表A类的java.lang.Class类的对象，而new所产生的对象，是依靠A类的java.lang.class对象为模板产生的新的对象到堆区），这个对象将作为程序访问方法区中的这些类型数据的外部接口。加载阶段与连接阶段的部分内容是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始。

## 2 验证

验证目的是为了确保Class文件的字节流中的信息不会危害到虚拟机，在该阶段主要完成以下四种验证：

1. 文件格式验证：验证字节流是否符合Class文件的规范，如主次版本号是否在当前虚拟机范围内，常量池中的常量是否有不被支持的类型。
2. 元数据验证：对字节码描述的信息进行语义分析，如这个类中是否有父类，是否集成了不被继承的类等。
3. 字节码验证：是整个验证过程中最复杂的一个阶段，通过验证数据流和控制流的分析，确定程序语义是否正确，主要针对方法体的验证。如：方法中的类型转换是否正确，跳转指令是否正确等。
4. 符号引用验证：这个动作在后面的解析过程中发生，主要是为了确保解析动作能正确执行。

## 3 准备

准备阶段是为类的静态变量分配内存并将其初始化为默认值，这些内存都将在方法区中进行分配。准备阶段不分配类中的实例变量的内存，实例变量将会在对象实例化时随着对象一起分配在Java堆中。如果该变量被final修饰，将在编译时生成ConstantValue，这样在准备阶段将直接设置成该初值。
```java
public static int value=123;//在准备阶段value初始值为0，初始化阶段才变为123。
```

### 4 解析

该阶段主要完成符号引用到直接引用的转换动作。解析动作并不一定在初始化动作完成之前，也有可能在初始化之后。

* 解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。其实就是将堆内存空间里的静态变量符号修改为已经申请了空间的静态变量地址的过程
  * 符号引用：（Symbolic References）符号引用以一组符号来描述所引用的目标，可以是任何形式的字面量，引用的目标并不一定已经加载到内存中，与虚拟机内存布局无关。
  * 直接引用：（Direct References）直接引用可以是直接指向目标的指针，相对偏移量，或是一个能间接定位到目标的句柄。与虚拟机内存布局相关。



### 5 初始化

初始化时类加载的最后一步，前面的类加载过程，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码。

初始化阶段是执行类构造器<clinit>()方法的过程

1. <clinit>()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并而成。编译器收集的顺序和语句在源文件中出现的顺序一致，静态语句块中只能访问到定义在它之前的变量，定义在它之后的变量，只能赋值，不能访问
2. <clinit>()方法与类的构造函数<init>()不同，不需要显式的调用父类构造器，虚拟机会保证父类的<clinit>()在子类的之前完成。因此，虚拟机执行的第一个<clinit>()方法肯定是java.lang.Object.
3. 由于父类<clinit>()方法先执行，也就意味着父类中定义的静态语句要优先于子类的变量赋值操作。
4. <clinit>()方法并不是必须的，如果一个类没有静态语句块也没有对变量赋值操作，就不会生成
5. 接口中不能使用静态语句块，但仍有变量初始化赋值的操作，因此也会生成<clinit>()方法，但与类不同的是，接口的<clinit>()方法不需要执行父接口的<clinit>()方法。只有当父几口中定义的变量被使用时，父接口才初始化，另外，接口的实现类在初始化时一样不会执行接口的<clinit>()方法。
6. 虚拟机会保证一个类的<clinit>()方法在多线程环境中正确的加锁同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都会阻塞，直到该方法执行完，如果在一个类的<clinit>()方法中有耗时很长的操作，可能会造成多个进程阻塞，在实际应用中，这种阻塞往往很隐蔽。

### 触发初始化

虚拟机规范严格规定了有且只有四种情况必须对类进行初始化（加载，验证，准备自动在之前开始）

1. 遇到new,getstatic,putstatic,invokestatic这4条字节码指令时，如果类没有进行初始化，则先初始化。这4个字节码常见的出现场景是：使用new关键字实例化对象的时候，读取或设置静态字段（被final修饰，已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
2. 反射调用时
3. 初始化一个类时，如果其父类还未初始化，则先出发父类初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类，虚拟机会先初始化这个主类


这4种情况称为对类的主动引用，其他情况称为被动引用。一下四种情况不会触发初始化

1. 对于访问静态字段，只有直接定义这个字段的类才被初始化，因此通过子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。但是对于HOTSPOT,会触发子类的加载。
2. 通过数组定义引用类，不会触发此类的初始化。
3. 常量在编译阶段会存入调用类的常量池，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。
4. 接口的加载和类加载过程稍有不同，接口不能有static代码段，但接口中还是会生成<clinit>()类构造器，用于初始化接口中所定义的成员变量。 一个接口在初始化时，并不要求其父类也初始化了。

###  补充说明
* 只有当某个类初始化之后，才会调用类的静态代码块。
* 初始化过程一方面是唯一的，另一方面是线程安全的。所以通过静态语句块的单例模式非常合理。