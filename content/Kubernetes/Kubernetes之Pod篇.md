
## Pre
**Q**:使用容器的“最佳实践”要求容器中只运行一个进程，是不是容器中只能运行一个进程呢？

**A**：容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的 init 进程或者 systemd 那样拥有进程管理的功能。比如，你的应用是一个 Java Web 程序（PID=1），然后你执行 docker exec 在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？因此，只有当容器中只存在一个进程时，才能对容器内的进程进行有效的管理。


**Q**:为什么需要Pod？

**A**:
1. 有些容器之间有着一种“超亲密关系”。这些具有“超亲密关系”的容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。这些容器有必要部署在同一环境中
2. 有些容器需要与其他容器共享Network或Volume等，例如B需要共享A的Network。但这样就会出现B依赖与A的情况，容器之间从对等关系退化为拓扑关系。如果将Network以及Volume定义在容器之上（也就是Pod）的一个级别，同一Pod中的容器共享Network和Volume，那容器之间还是单纯的对等关系。


**Pod是Kubernetes中最小的编排单位**, 可以类比与传统部署环境中的“虚拟机”。**凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的**。这些属性的共同特征在于，他们都是对“机器”进行描述。

## Pod中常用的字段
- nodeSelector.供用户将Pod与Node进行绑定。

用法示例：
```yaml
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```
这代表这个Pod只能运行在携带了`disktype:ssd`标签的节点上。

- nodeName. 
一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到。

- hostAliases. 定义Pod的hosts文件
```yaml
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```
这样，这个Pod启动后，/etc/hosts中就会有如下配置:
```text
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

需要注意的是，在kubernetes中，如果想要配置hosts文件，必须使用这种方式，否则在Pod重建后，hosts文件会被覆盖。

**Q**:Pod的生命周期
**A**:
1. Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
2. Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
3. Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
4. Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
5. Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。