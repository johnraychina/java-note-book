# Java 9 Modules

https://www.oracle.com/corporate/features/understanding-java-9-modules.html

http://openjdk.java.net/projects/jigsaw/quick-start


照例按照5W2H的套路提出问题：
- 是什么？
- 为什么？
- 如何使用？


## Java模块化是什么？
模块化定义了一个比package更高级别的聚合层级。

关键在于新引入的语言要素 module, 包括：

唯一命名、一组相关的package，以及资源（图片、xml等），以及一个module-info.class文件描述。



## Java为什要模块化？
java应用的体积问题（嵌入式）

目标：
- Reliable configuration: 明确的依赖关系，可控的类加载。
- Strong encapsulation: 强封装
- Scalable Java platform: 你可以定制你的java运行时，只打包你需要的模块。
- Greater platform integrity: 把原本无意暴露给用户的内部API，封装起来。
- Improved performance: 性能提升，只需要检查用到的最小依赖。

Readability, Accessibility

## 如何使用？

使用jlink 打包


显示JDK模块
```sh
java --list-modules
```

创建 module-info.java 
```java
module my_module {
    requires modulename;
    requires transitive modulename; //依赖传递，应用my_module模块时，会把这个模块也引入进来
    exports com.zhangyi.math; //指定暴露的包
    exports com.zhangyi.dl to outer.package; // 指定给谁看

    uses javax.naming.ldap.LdapDnsProvider; //用到了什么接口，需要别人来是实现
    provides xxx.Provider with xxx.MyProvider;//作为一个接口实现实现

    open ; //允许反射接口 查看本模块的包
    opens my_package1; //允许反射接口 查看本模块指定包
    opens xxx to yyy;  //允许指定的包，用反射接口 查看本模块的指定包
}
```
