**Q**: DaemonSet的的作用
**A**: 在 Kubernetes 集群里，运行一个 Daemon Pod。 这个 Pod 有如下三个特征：
1. 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
2. 每个节点上只有一个这样的 Pod 实例；
3. 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

**Q**: DaemonSet的应用实例
**A**:
1. 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
2. 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。

DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早

**Q**: DaemonSet 如何保证每个 Node 上有且只有一个被管理的 Pod
**A**:
DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 name=fluentd-elasticsearch 标签的 Pod 在运行。而检查的结果，可能有这么三种情况：
1. 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
2. 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；
3. 正好只有一个这种 Pod，那说明这个节点是正常的。其中，删除节点（Node）上多余的 Pod 非常简单，直接调用 Kubernetes API 就可以了。
DaemonSet通过污点容忍机制选择node

相比于 Deployment，DaemonSet 只管理 Pod 对象，然后通过 nodeAffinity 和 Toleration 这两个调度器的小功能，保证了每个节点上有且只有一个 Pod。

DaemonSet 使用 ControllerRevision，来保存和管理自己对应的“版本”。

