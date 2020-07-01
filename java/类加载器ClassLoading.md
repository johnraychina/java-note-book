
# 如何卸载class ?

Perm OOM问题
Groovy类加载和卸载


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


# Pandora 中间件类加载器，如何做到app class与中间件的class隔离与连接？

# SOFA 中间件类加载器，如何做到app class与中间件的class隔离与连接？

# OSGI 类加载器如何做到可控的class的隔离和连接？


ClassLoader类加载器与SecurityManager


# endorsed standards override feature for Java <= 8 

# and the upgradeable modules feature for Java 9+.


























