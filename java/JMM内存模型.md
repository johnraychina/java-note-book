https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html

内存模型描述了：给定程序  与 它的执行路径，判断执行路径是否合法。
Java内存模型在每个执行中，根据一定规则，检查执行中的read是否有效，检查被read观察到的write是否有效。



17.4. Memory Model
A memory model describes, given a program and an execution trace of that program, whether the execution trace is a legal execution of the program. The Java programming language memory model works by examining each read in an execution trace and checking that the write observed by that read is valid according to certain rules.

The memory model describes possible behaviors of a program. An implementation is free to produce any code it likes, as long as all resulting executions of a program produce a result that can be predicted by the memory model.

This provides a great deal of freedom for the implementor to perform a myriad of code transformations, including the reordering of actions and removal of unnecessary synchronization.