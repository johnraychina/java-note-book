# Java parallel stream

java stream的collect/any等操作触发evalute求值操作

# 怎样将任务丢ForkJoinPool中，怎样收集回来？
ForkJoinTask

# 任务队列会OOM吗？
## 保护措施1：ForkJoinPool.WorkQueue#MAXIMUM_QUEUE_CAPACITY
```java
if (size > MAXIMUM_QUEUE_CAPACITY)
                throw new RejectedExecutionException("Queue capacity exceeded");
```
## 保护措施2：有限线程数 + 阻塞
workQueues 队列数
array 任务数组

q.growArray()

# 提交任务会被执行慢的任务阻塞吗？
会的，而且main线程会阻塞知道所有线程join
evaluate求值操作触发执行流管道 的 evaluateParallel并行操作，分多线程拉取执行任务。


# 队列状态是怎么样的？


# 提交任务
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at java.util.concurrent.ForkJoinPool.externalPush(ForkJoinPool.java:2419)
	  at java.util.concurrent.ForkJoinTask.fork(ForkJoinTask.java:702)
	  at java.util.stream.AbstractTask.compute(AbstractTask.java:313)
	  at java.util.concurrent.CountedCompleter.exec(CountedCompleter.java:731)
	  at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
	  at java.util.concurrent.ForkJoinTask.doInvoke(ForkJoinTask.java:401)
	  at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:734)
	  at java.util.stream.ReduceOps$ReduceOp.evaluateParallel(ReduceOps.java:714)
	  at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
	  at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
	  at com.zhangyi.StreamOperations.main(StreamOperations.java:51)


# 执行任务
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at com.zhangyi.StreamOperations.lambda$main$0(StreamOperations.java:49) //执行任务
	  at com.zhangyi.StreamOperations$$Lambda$1.2101440631.accept(Unknown Source:-1) 
	  at java.util.stream.IntPipeline$10$1.accept(IntPipeline.java:361)
	  at java.util.stream.Streams$RangeIntSpliterator.forEachRemaining(Streams.java:110)
	  at java.util.Spliterator$OfInt.forEachRemaining(Spliterator.java:693)
	  at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
	  at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:471)
	  at java.util.stream.ReduceOps$ReduceTask.doLeaf(ReduceOps.java:747)
	  at java.util.stream.ReduceOps$ReduceTask.doLeaf(ReduceOps.java:721)
	  at java.util.stream.AbstractTask.compute(AbstractTask.java:316)        //执行任务start
	  at java.util.concurrent.CountedCompleter.exec(CountedCompleter.java:731)
	  at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
	  at java.util.concurrent.ForkJoinTask.doInvoke(ForkJoinTask.java:401)
	  at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:734)
	  at java.util.stream.ReduceOps$ReduceOp.evaluateParallel(ReduceOps.java:714)
	  at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
	  at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
	  at com.zhangyi.StreamOperations.main(StreamOperations.java:58)
      "ForkJoinPool.commonPool-worker-6@784" daemon prio=5 tid=0x12 nid=NA runnable

  java.lang.Thread.State: RUNNABLE
	 blocks main@1
	  at java.util.Collections$SynchronizedCollection.add(Collections.java:2035)
	  - locked <0x32e> (a java.util.Collections$SynchronizedSet)
	  at java.lang.ClassLoader.checkPackageAccess(ClassLoader.java:508)
	  at com.zhangyi.StreamOperations.lambda$main$0(StreamOperations.java:49)
	  at com.zhangyi.StreamOperations$$Lambda$1.2101440631.accept(Unknown Source:-1)
	  at java.util.stream.IntPipeline$10$1.accept(IntPipeline.java:361)
	  at java.util.stream.Streams$RangeIntSpliterator.forEachRemaining(Streams.java:110)
	  at java.util.Spliterator$OfInt.forEachRemaining(Spliterator.java:693)
	  at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
	  at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:471)
	  at java.util.stream.ReduceOps$ReduceTask.doLeaf(ReduceOps.java:747)
	  at java.util.stream.ReduceOps$ReduceTask.doLeaf(ReduceOps.java:721)
	  at java.util.stream.AbstractTask.compute(AbstractTask.java:316)
	  at java.util.concurrent.CountedCompleter.exec(CountedCompleter.java:731)
	  at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
	  at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1056)
	  at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1692)
	  at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:157)
