
# SpringBoot 怎么启动和做类加载
https://www.atatech.org/articles/45367

## spring boot应用打包：
spring boot maven插件生成fat jar包，里面包含：
  META-INF/MANIFEST.MF:
    Main-Class 指向启动类JarLauncher
    Start-Class指向应用的main方法类
  应用的资源：class, 配置文件等
  lib目录：应用依赖的jar包
  org/springframework/loader: 类加载器

## 启动：JarLauncher获取lib/下面的jar，并创建一个LaunchedURLClassLoader
LaunchedURLClassLoader和普通的URLClassLoader的不同之处是，它提供了从Archive里加载.class的能力。

## 总结
spring boot应用打包之后，生成一个fat jar，里面包含了应用依赖的jar包，还有Spring boot loader相关的类
Fat jar的启动Main函数是JarLauncher，它负责创建一个LaunchedURLClassLoader来加载/lib下面的jar，并以一个新线程启动应用的Main函数。


# Pandora 如何做类加载
目的：中间件之间类隔离，app的类不影响中间件，中间件的部分类对app可见



- 应用app类加载器 bizClassLoader:ReLanchURLClassLoader -> ExtClassLoader

- Pandora容器类加载器 pandoraClassLoader:UrlClassLoader -> ExtClassLoader

- 每个中间件插件有一个类加载器 ModuleClassLoader ，所以中间件之间互不影响，也不受应用app影响。

- 导出：中间件的类被ModuleClassLoader加载后，会被<u>选择性的导出</u>给应用app类加载器。


1. pandora boot的应用app被打包，MANIFEST文件指定Main-Class:SarLauncher(继承自JarLauncher)

2. SarLauncher会调用MANIFEST文件指定Start-Class:Application(应用app入口)

3. Application中调用PandoraBootstrap.run 创建应用app类加载器 bizClassLoader(ReLanchURLClassLoader) 

    3.1 扫描中间件类 <br>
    每个中间件按规范里面有一个特殊的properties文件标明中间件名称和版本
    应用app的lib下有一堆中间件jar，扫描到这个特殊命名的的文件，然后登记中间件jar包url，然后加载PandoraContainer，反射调用PandoraContainer.start(sarUrl, pluginUrls, bizClassLoader)来启动中间件

    3.2 PandoraContainer加载中间类并导出 <br>
     将pluginUrls指定的中间件jar包用ModuleClassLoader加载启动中间件，再导出给 bizClassLoader

4. 完成

## 小结
    从bizClassLoader 里查找到所有的pandora plugin jar。 
    从sar包里查找lib目录下的jar，并构造 PandoraContainer调用ModuleClassLoader。
    再返回 exportedClasses 缓存。


# ClassLoader类加载器与SecurityManager


# endorsed standards override feature for Java <= 8 

# and the upgradeable modules feature for Java 9+.

# 一个类是何时会被加载？
当运行中需要这个类的时候：
- 执行时引用： 遇到new, getstatic, putstatic等指令时
- 对类进行反射调用
- 初始化某个子类时
- main方法的启动类
- 使用jdk1.7 动态语言支持的时候

# 一个类是如何被加载进来的？
ClassLoader loadClass，加载，解析，验证，注册到 Dictionary

一个class被加载后，他的唯一标志符为 full class name + classloader

## 双亲委派机制
默认情况下


## 打破双亲委派的几个场景
JavaEE 容器 

SPI类加载

OSGI 导入导出类

# Class unloading卸载
在网上搜索 Perm OOM问题，Groovy OOM，你就会发现有许多类加载和卸载的问题。

平常我们不会关心类的卸载，尤其是java8之后默认用MetaSpace（堆外存储，仅受实际内存限制）。

可一旦你遇到动态加载类的情况时，事情就变得复杂起来，你就不得不考虑类是如何卸载的。

## 执行class unloading
时机：看jdk版本和设置，在PermSpace或者MetaSpace空间不够时触发FullGC时

范围：当GC Roots不可达时才可能被GC掉：即class的instance和class本身没有被引用时 。

https://www.dynatrace.com/resources/ebooks/javabook/how-garbage-collection-works/

那么什么是GC Roots呢？ 可从堆外（non-heap）引用的对象：

- 活动的线程 Active Java threads are always considered live objects and are therefore GC roots. 

- 线程本地变量 Local variables are kept alive by the stack of a thread.

- 类静态变量 Static variables are referenced by their classes.

- JNI本地方法引用 JNI References



# Tomcat ClassLoading 类加载

需要实现3个基本目标：
1、基本的javase的类要用BootStrapClassLoader加载
2、tomcat server本身的class要与web app的隔离， 并做有限共享
3、web app之间互相隔离， 并做有限共享


## 概要
      Bootstrap
          |
       System
          |
       Common
       /     \
  Webapp1   Webapp2 ...

BootstrapClassLoader 加载JVM基础类
SystemClassLoader 注意：与普通应用从CLASSPATH加载不同，tomcat按自己的catlina.bat/sh启动脚本定义加载bootstrap.jar, tomcat-juli.jar日志, commons-deamnon.jar
CommonClassLoader，这个ClassLoader加载的类是给Tomcat内部和webapp共用的，它会加载catalina.proerties文件中common.loader指定的路径，默认是： $CATALINA_BASE/lib和$CATALINA_HOME/lib，里面有catalina，jasper(jsp)，servlet-api，websocket等jar包
WebappClassLoader 加载webpp各自的/WEB-INF/classes, /WEB-INF/lib类，对其他webpp不可见


一个WebappClassLoader收到一个class加载请求时，它是这么加载的：
- **首先** 从本地类路径下搜索
但是也有例外：JRE 基础类仍然走它parent加载，
其中又有例外：XML parser组件又可以利用JVM的机制覆盖（Java8和之前可以用启动参数endorse机制，java9+走upgradeable modules机制）

- **最后** JavaEE API类的加载总是先由WebAppClassLoader来代理，Tomcat中其他class loaders都走普通的代理模式。

总结一下：
从web app的角度看，一个class或者resource搜索遵循以下顺序：
- JVM Bootstrap classes 
- web app /WEB-INF/classes  
- web app /WEB-INF/lib classes
- System ClassLoader classes
- Common ClassLoader classes

如果AppClassLoader配置为了delegate=true(默认是false)，那么搜索顺序就会变为：
- JVM Bootstrap classes
- SystemClassLoader classes
- CommonClassLoader classes
- web app /WEB-INF/classes
- web app /WEB-INF/lib


## 高级配置

  Bootstrap
      |
    System
      |
    Common
     /  \
Server  Shared
         /  \
   Webapp1  Webapp2 ..




























