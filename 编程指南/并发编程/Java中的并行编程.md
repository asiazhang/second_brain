# Java中的并行编程

## parallelStream
本质上是ForkJoin线程池。

## 线程池
通过submit提交任务。

### 普通线程池
有异常的任务不影响整个执行流程。

### ForkJoin线程池
失败的任务会中断整个执行流程。