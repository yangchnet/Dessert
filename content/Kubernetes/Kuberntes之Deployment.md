Deployment是Kubernetes中的一个控制器，负责对pod进行控制

**Q**:控制器对对象进行控制的逻辑是什么样的
**A**:通过`控制循环`对API对象进行控制，其控制逻辑用伪代码表示如下：
```go
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```
在具体实现时，实际状态来自Kubernetes本身，而期望状态来自用户提交的Yaml文件。这个操作，通常被叫作调谐（Reconcile）。这个调谐的过程，则被称作“Reconcile Loop”（调谐循环）或者“Sync Loop”（同步循环）。

**Q**:对于Deployment，其创建出的Pod的ownerReference是谁，是Deployment本身吗
**A**:




