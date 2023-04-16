
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
**A**:Deployment操作的是ReplicaSet对象。在Pod进行水平伸缩时，Deployment就可以完成任务（但其实不是这样的，这里只是探讨一种可能性）；但要想做到`滚动更新`，光有Deployment是不行的，因为Deployment的template字段决定了其只能管理一种配置的Pod。Deployment管理ReplicaSet，ReplicaSet管理Pod，这样如果需要进行滚动更新，Deployment只需创建新的ReplicaSet，然后逐步下线原有的ReplicaSet即可。
![20230411204357](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20230411204357.png)
![20230411204624](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20230411204624.png)


一个应用的版本，对应的正是一个ReplicaSet，这个版本应用的 Pod 数量，则由 ReplicaSet 通过它自己的控制器（ReplicaSet Controller）来保证。
而Deployment，其实就是对“应用”的抽象，我们可以使用这个 Deployment 对象来描述应用，使用 kubectl rollout 命令控制应用的版本。

但Deployment只适合那些较简单的应用。




