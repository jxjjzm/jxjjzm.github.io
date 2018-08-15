### Jvm之虚拟机类加载机制 ###
***

### 一、Class文件基本组织结构 ###

作为Java程序猿，我们知道，我们写好的Java源文件，最后会被Java编译器编译成后缀为.class的文件，该类型的文件是由字节组成的文件，又叫字节码文件。那么，class字节码文件里面到底是有什么呢？它又是怎样组织的呢？本文重点针对虚拟机类加载机制做讲解，再此不深究class文件基本组织结构，如需详细了解可以参考《Java虚拟机原理图解》系列博文： [《Java虚拟机原理图解》](http://blog.csdn.net/u010349169/article/category/2620885)


### 二、类加载 ###

虚拟机类加载机制就是虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、装换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型。更通俗地来说类加载就是将类的.class文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个java.lang.Class对象，用来封装类在方法区内的数据结构。类的加载的最终产品是位于堆区中的Class对象，Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。


#### 1.类的生命周期 ####

类从加载到虚拟机内存中开始到，到卸载出内存为止，它的整个生命周期包括了：加载（Loading）、验证（Verification）、准备(Preparation)、解析（Resolution）、初始化（Initialization）、使用(Using)、 卸载（Unloading）。如图所示：


![类的生命周期](http://www.ityouknow.com/assets/images/2017/jvm/class.png?_=6482464)

其中类加载的过程包括了加载、验证、准备、解析、初始化五个阶段。在这五个阶段中，加载、验证、准备和初始化这四个阶段发生的顺序是确定的，而解析阶段则不一定，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java语言的运行时绑定（也成为动态绑定或晚期绑定）。另外注意这里的几个阶段是按顺序开始，而不是按顺序进行或完成，因为这些阶段通常都是互相交叉地混合进行的，通常在一个阶段执行的过程中调用或激活另一个阶段。

结束生命周期——在如下几种情况下，Java虚拟机将结束生命周期：


- 执行了System.exit()方法
- 程序正常执行结束
- 程序在执行过程中遇到了异常或错误而异常终止
- 由于操作系统出现错误而导致Java虚拟机进程终止



#### 2.类加载过程 ####



- ***加载***：查找并加载类的二进制数据。加载是虚拟机类加载过程的第一个阶段，在加载阶段，虚拟机需要完成以下三件事情：
	- （1）通过一个类的全限定名来获取其定义的二进制字节流。
	- （2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
	- （3）在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。

相对于类加载的其他阶段而言，加载阶段（准确地说，是加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，因为开发人员既可以使用系统提供的类加载器来完成加载，也可以自定义自己的类加载器来完成加载。加载阶段完成后，虚拟机外部的 二进制字节流就按照虚拟机所需的格式存储在方法区之中，而且在Java堆中也创建一个java.lang.Class类的对象，这样便可以通过该对象访问方法区中的这些数据。




- ***验证***：确保被加载的类的正确性。验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。验证阶段大致会完成4个阶段的检验动作：
	- **（1）文件格式验证**：验证字节流是否符合Class文件格式的规范；例如：是否以魔数0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
	- **（2）元数据验证**：对字节码描述的信息进行语义分析（注意：对比javac编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了java.lang.Object之外。
	- **（3）字节码验证**：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
	- **（4）符号引用验证**：确保解析动作能正确执行。

验证阶段是非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用-Xverifynone参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

 


- ***准备***：为类的静态变量分配内存，并将其初始化为默认值准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。对于该阶段有以下几点需要注意：
	- （1）这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
	- （2）这里所设置的初始值通常情况下是数据类型默认的零值（如0、0L、null、false等），而不是被在Java代码中被显式地赋予的值。
	- （3）如果类字段的字段属性表中存在ConstantValue属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。
 




- ***解析***：把类中的符号引用转换为直接引用。解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。
	- 符号引用就是一组符号来描述目标，可以是任何字面量。
	- 直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。




- ***初始化***：
 前面过程都是以虚拟机主导，而初始化阶段开始执行类中的 Java 代码。JVM负责对类进行初始化，主要对类变量进行初始化。在Java中对类变量进行初始值设定有两种方式：①声明类变量时指定初始值；②使用静态代码块为类变量指定初始值

 JVM初始化步骤：

- 1、假如这个类还没有被加载和连接，则程序先加载并连接该类
- 2、假如该类的直接父类还没有被初始化，则先初始化其直接父类
- 3、假如类中有初始化语句，则系统依次执行这些初始化语句

类初始化时机：只有当对类的主动使用的时候才会导致类的初始化(而加载、验证、准备自然需要在此之前完成)，以下五种情况必须对类进行初始化，即类的主动使用：

1. 遇到 new、getstatic、putstatic 或 invokestatic 这 4 条字节码指令时没初始化触发初始化。使用场景：使用 new 关键字实例化对象、读取一个类的静态字段(被 final 修饰、已在编译期把结果放入常量池的静态字段除外)、调用一个类的静态方法。
2. 使用 java.lang.reflect 包的方法对类进行反射调用的时候。
3. 当初始化一个类的时候，如果发现其父类还没有进行初始化，则需先触发其父类的初始化。
4. 当虚拟机启动时，用户需指定一个要加载的主类(包含 main() 方法的那个类)，虚拟机会先初始化这个主类。
5. 当使用 JDK 1.7 的动态语言支持时，如果一个 java.lang.invoke.MethodHandle 实例最后的解析结果 REF_getStatic、REF_putStatic、REF_invokeStatic 的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需先触发其初始化。
 



#### 3.类加载器 ####

一、类加载器分类

站在Java虚拟机的角度来讲，只存在两种不同的类加载器：启动类加载器：它使用C++实现（这里仅限于Hotspot，也就是JDK1.5之后默认的虚拟机，有很多其他的虚拟机是用Java语言实现的），是虚拟机自身的一部分；所有其他的类加载器：这些类加载器都由Java语言实现，独立于虚拟机之外，并且全部继承自抽象类java.lang.ClassLoader，这些类加载器需要由启动类加载器加载到内存中之后才能去加载其他的类。

站在Java开发人员的角度来看，类加载器可以大致划分为以下三类：



- ***启动类加载器：Bootstrap ClassLoader***，负责加载存放在JDK\jre\lib(JDK代表JDK的安装目录，下同)下，或被-Xbootclasspath参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar，所有的java.*开头的类均被Bootstrap ClassLoader加载）。启动类加载器是无法被Java程序直接引用的。



- ***扩展类加载器：Extension ClassLoader***，该加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载DK\jre\lib\ext目录中，或者由java.ext.dirs系统变量指定的路径中的所有类库（如javax.*开头的类），开发者可以直接使用扩展类加载器。



- ***应用程序类加载器：Application ClassLoader***，该类加载器由sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。


二、JVM类加载机制

- 全盘负责，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入；
- 父类委托，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类
- 缓存机制，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效


类的加载——类加载有三种方式：

- 1）、命令行启动应用时候由JVM初始化加载
- 2）、通过Class.forName()方法动态加载
- 3）、通过ClassLoader.loadClass()方法动态加载

（

Class.forName()和ClassLoader.loadClass()区别：

- Class.forName()：将类的.class文件加载到jvm之外，还会对类进行解释，执行类中的static块；
- ClassLoader.loadClass()：只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。

**注**：Class.forName(name, initialize, loader)带参函数也可控制是否加载static块。并且只有调用了newInstance()方法采用调用构造函数，创建类的对象 。

）

**一）、双亲委派模型------> 默认的模式**


![](http://images2015.cnblogs.com/blog/331425/201606/331425-20160621125943787-249179344.jpg)

**注意：这里父类加载器并不是通过继承关系来实现的，而是采用组合实现的。**

双亲委派模型的工作流程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

双亲委派机制：

1、当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。

2、当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。

3、如果BootStrapClassLoader加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用ExtClassLoader来尝试加载；

4、若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。

ClassLoader源码分析：

	public Class<?> loadClass(String name)throws ClassNotFoundException {
            return loadClass(name, false);
    }
    
    protected synchronized Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException {
            // 首先判断该类型是否已经被加载
            Class c = findLoadedClass(name);
            if (c == null) {
                //如果没有被加载，就委托给父类加载或者委派给启动类加载器加载
                try {
                    if (parent != null) {
                         //如果存在父类加载器，就委派给父类加载器加载
                        c = parent.loadClass(name, false);
                    } else {
                    //如果不存在父类加载器，就检查是否是由启动类加载器加载的类，通过调用本地方法native Class findBootstrapClass(String name)
                        c = findBootstrapClass0(name);
                    }
                } catch (ClassNotFoundException e) {
                 // 如果父类加载器和启动类加载器都不能完成加载任务，才调用自身的加载功能
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }


双亲委派模型意义：

- 系统类防止内存中出现多份同样的字节码
- 保证Java程序安全稳定运行




**二）、破坏双亲委派模型**

双亲模式是默认的模式，但不是必须这么做，如：
- Tomcat的WebappClassLoader就会先加载自己的class,找不到再委托parent
- OSGI的ClassLoader形成网状结构，根据需要自由加载Class 

以Tomcat为例：

一个功能健全的WEB服务器都要解决如下几个问题：

- 部署在同一个服务器上的两个web应用程序所使用的Java类库可以实现相互隔离。
- 部署在同一个服务器上的两个web应用程序所使用的Java类库可以实现相互共享。
- 服务器需要尽可能的保证自身的安全不受部署的Web应用程序影响。
- 支持JSP应用的web服务器，十有八九都需要支持HotSwap功能。

在Tomcat的目录结构中，有三组目录（"/common/*","/server/*","/shared/*"）可以存放Java类库，另外还可以加上Web应用程序自身的目录"/WEB-INF/*",一共四组，把Java类库放置在这些目录的含义分别是：

- 放置在/common目录中：类库可被Tomcat和所有的Web应用程序共同使用。
- 放置在/server目录中：类库可被Tomcat使用，对所有的web应用程序都不可见。
- 放置在/shared目录中：类库可被所有的web应用程序共同使用，但对Tomcat自己不可见。
- 放置在/WebApp/WEB_INF目录中：类库仅仅被次Web应用程序使用，对Tomcat和其他Web应用程序都不可见。

为了支持这套目录结构，并对目录里面的类库进行加载和隔离，Tomcat自定义了多个类加载器：

![](http://upload-images.jianshu.io/upload_images/6715251-b21326fe843cce9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



三、自定义类加载器

通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。比如应用是通过网络来传输 Java 类的字节码，为保证安全性，这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现。自定义类加载器一般都是继承自 ClassLoader 类，从上面对 loadClass 方法来分析来看，我们只需要重写 findClass 方法即可。

自定义类加载器的好处：

- 1）在执行非置信代码之前，自动验证数字签名。
- 2）动态地创建符合用户特定需要的定制化构建类。
- 3）从特定的场所取得java class，例如数据库中和网络中。

	

		public class MyClassLoader extends ClassLoader {

	    	private String root;
	
	    	protected Class<?> findClass(String name) throws ClassNotFoundException {
	        	byte[] classData = loadClassData(name);
	        	if (classData == null) {
	            	throw new ClassNotFoundException();
	        	} else {
	            	return defineClass(name, classData, 0, classData.length);
	        	}
	    	}
	
	    	private byte[] loadClassData(String className) {
	        	String fileName = root + File.separatorChar+ className.replace('.', File.separatorChar) + ".class";
		        try {
		            InputStream ins = new FileInputStream(fileName);
		            ByteArrayOutputStream baos = new ByteArrayOutputStream();
		            int bufferSize = 1024;
		            byte[] buffer = new byte[bufferSize];
		            int length = 0;
		            while ((length = ins.read(buffer)) != -1) {
		                baos.write(buffer, 0, length);
		            }
		            return baos.toByteArray();
		        } catch (IOException e) {
		            e.printStackTrace();
		        }
	        	return null;
	    	}
	
		    public String getRoot() {
		        return root;
		    }
		
		    public void setRoot(String root) {
		        this.root = root;
		    }
	
		    public static void main(String[] args)  {
		
		        MyClassLoader classLoader = new MyClassLoader();
		        classLoader.setRoot("E:\\temp");
		
		        Class<?> testClass = null;
		        try {
		            testClass = classLoader.loadClass("com.neo.classloader.Test2");
		            Object object = testClass.newInstance();
		            System.out.println(object.getClass().getClassLoader());
		        } catch (ClassNotFoundException e) {
		            e.printStackTrace();
		        } catch (InstantiationException e) {
		            e.printStackTrace();
		        } catch (IllegalAccessException e) {
		            e.printStackTrace();
		        }
		    }
		}



### 附录 ###

- [深入理解Java类加载器(ClassLoader)](https://blog.csdn.net/javazejian/article/details/73413292)















































































































































































