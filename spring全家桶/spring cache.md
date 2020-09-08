
spring cache机制：

通过注解 + 切面拦截按注解的配置策略读写缓存



## 读操作
读的时候，先执行lookup查缓存，缓存miss则执行业务逻辑，最后再put缓存

核心逻辑在CacheAspectSupport#execute
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at com.zhangyi.FusionCache.lookup(FusionCache.java:40)
	  at org.springframework.cache.support.AbstractValueAdaptingCache.get(AbstractValueAdaptingCache.java:58)
	  at org.springframework.cache.interceptor.AbstractCacheInvoker.doGet(AbstractCacheInvoker.java:73)
	  at org.springframework.cache.interceptor.CacheAspectSupport.findInCaches(CacheAspectSupport.java:525)
	  at org.springframework.cache.interceptor.CacheAspectSupport.findCachedItem(CacheAspectSupport.java:490)
	  at org.springframework.cache.interceptor.CacheAspectSupport.execute(CacheAspectSupport.java:372)
	  at org.springframework.cache.interceptor.CacheAspectSupport.execute(CacheAspectSupport.java:316)
	  at org.springframework.cache.interceptor.CacheInterceptor.invoke(CacheInterceptor.java:61)
	  at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:185)
	  at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:688)
	  at com.zhangyi.SalesmanComponent.idOf(<generated>:-1)

可以看到idOf方法被CacheAspectSupport拦截，做了如下处理：

- 1、根据@Cacheable的cacheNames找cache
而cache是继承了AbstractValueAdaptingCache

- 2、从cache中get(key)
注意观察返回参数ValueWrapper是一个包装类，（这样cache中才能实现存放null value）

有时候明明新增了数据，但是查不到数据，就是因为cache中缓存了null并且没过期。

分布式系统中，如果使用本地缓存，往往容易出现这种一致性问题。所以需要通知机制：可以利用redis pub/sub或者MQ广播。


- 3、如果没get到，会根据EL表达式判断是否要put到缓存中
collectPutRequests -> isConditionPassing

如果要put，就会调用org.springframework.cache.Cache#put


## 写操作
先执行业务逻辑，然后执行缓存evict

"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at com.zhangyi.FusionCache.evict(FusionCache.java:80)
	  at org.springframework.cache.interceptor.AbstractCacheInvoker.doEvict(AbstractCacheInvoker.java:100)
	  at org.springframework.cache.interceptor.CacheAspectSupport.performCacheEvict(CacheAspectSupport.java:466)
	  at org.springframework.cache.interceptor.CacheAspectSupport.processCacheEvicts(CacheAspectSupport.java:447)
	  at org.springframework.cache.interceptor.CacheAspectSupport.execute(CacheAspectSupport.java:404)
	  at org.springframework.cache.interceptor.CacheAspectSupport.execute(CacheAspectSupport.java:316)
	  at org.springframework.cache.interceptor.CacheInterceptor.invoke(CacheInterceptor.java:61)
	  at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:185)
	  at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:688)
	  at com.zhangyi.SalesmanComponent$$EnhancerBySpringCGLIB$$c7f8d4f4.onSalesmanDelete(<generated>:-1)

