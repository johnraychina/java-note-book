invokedynamic 使用与原理




支持应用级别的代码来确定执行哪个方法调用（执行时动态判断）


Lambda表达式

# Why

# What

与常规Instance Method和Static Method 


# How
https://www.infoq.com/articles/Java-8-Lambdas-A-Peek-Under-the-Hood/

## Method Handle 方法句柄
BSM(BootStrap Method)

Java Interpreter 字节码解释器遇到invokedynamic指令时，BSM会被调用，

它会返回一个CallSite对象，包含了MethodHandle，表明了调用点实际要执行哪个方法

* 对比反射：以一种安全、高效的方式来实现反射的核心功能，并尽可能保证类型安全。




## MethodType 
调用方法需要的四要素：
- 名称
- 签名（参数列表类型、返回值类型）
- 定义它的类
- 实现方法的字节码


## MethodHandles 工厂方法
MethodHandles.lookup() 得到Lookup上下文 ==> 寻找方法句柄MethodHandle


