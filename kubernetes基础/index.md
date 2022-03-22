# Kubernetes基础


<!--more-->

# Kubernetes 基础

## 一、架构

### 1.1 组件

**Master** ： scheduler、API-Server、rc（副本控制器）

**Etcd** ： 存储（键值对）（分布式）（持久化） v2:内存 v3: DB

**Node** ： kubelet（与容器运行时进行交互）、kube proxy（负载均衡，写入规则至 iptables）

**CoreDNS** ： 为集群中 svc 创建“域名-IP”的对应解析

**Ingress Controller** ： 实现七层的代理

## 二、Pod 状态与生命周期管理

### 2.3 Init 容器

#### (1) 探针

- init C 可以做服务探针，但是 init C 执行完之后就无法持续在线了

- 探针是由 kubelet 对容器执行的定期诊断，要执行的话，kubelet 需要调用容器实现的 handler

  - execAction ：在容器中执行指定命令，成功则诊断成功
  - TCPSocketAction：对容器对应 IP 的 tcp 端口进行检查，如果端口打开，则诊断成功
  - HTTPGetAction：执行七层 HTTP 请求，状态码满足要求则诊断成功

- 两种探测方案
  - livenessProbe : 存活探测 livenessProbe-exec ：执行命令
  - readinessProbe ：就绪探测 readinessProbe-httpget ： http 请求

## 三、集群资源管理

##### 资源清单类别

- 名称空间级别

  1. 工作负载型：Pod、RS、Deployment、Statefulset、DeamonSet、Job...
  2. 服务发现及负载均衡： Service、Ingress

- 集群级别（全局可见资源）

  - namespace、node、role、clusterRole

- 元数据型资源

  - HPA

### 3.1 Node

#### (1) 信息

![image-20220220225409792](/img/image-20220220225409792.png)

![image-20220220225442732](/img/image-20220220225442732.png)

#### (2) <span id="lease">Lease 内容</span>——节点心跳

Kubernetes 节点发送的心跳帮助你的集群确定每个节点的可用性，并在检测到故障时采取行动。

对于节点，有两种形式的心跳:

- 更新节点的 `.status`
- [Lease](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/) 对象 在 `kube-node-lease` [命名空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)中。 每个节点都有一个关联的 Lease 对象。

与 Node 的 `.status` 更新相比，`Lease` 是一种轻量级资源。 使用 `Leases` 心跳在大型集群中可以减少这些更新对性能的影响。

kubelet 负责创建和更新节点的 `.status`，以及更新它们对应的 `Lease`。

- 当状态发生变化时，或者在配置的时间间隔内没有更新事件时，kubelet 会更新 `.status`。 `.status` 更新的默认间隔为 5 分钟（比不可达节点的 40 秒默认超时时间长很多）。
- `kubelet` 会每 10 秒（默认更新间隔时间）创建并更新其 `Lease` 对象。 `Lease` 更新独立于 `NodeStatus` 更新而发生。 如果 `Lease` 的更新操作失败，`kubelet` 会采用指数回退机制，从 200 毫秒开始 重试，最长重试间隔为 7 秒钟。

- lease 资源全称 leases.coordination.k8s.io

```shell
# 获取lease资源
kubectl get leases.coordination.k8s.io --all-namespaces
```

#### (3) Condition

conditions 字段描述所有 Running 节点的状态。

![image-20220207111836091](/img/image-20220207111836091.png)

| 节点状况             | 描述                                                                                                                                                                                    |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Ready`              | 如节点是健康的并已经准备好接收 Pod 则为 `True`；`False` 表示节点不健康而且不能接收 Pod；`Unknown` 表示节点控制器在最近 `node-monitor-grace-period` 期间（默认 40 秒）没有收到节点的消息 |
| `DiskPressure`       | `True` 表示节点存在磁盘空间压力，即磁盘可用量低, 否则为 `False`                                                                                                                         |
| `MemoryPressure`     | `True` 表示节点存在内存压力，即节点内存可用量低，否则为 `False`                                                                                                                         |
| `PIDPressure`        | `True` 表示节点存在进程压力，即节点上进程过多；否则为 `False`                                                                                                                           |
| `NetworkUnavailable` | `True` 表示节点网络配置不正确；否则为 `False`                                                                                                                                           |

#### (4) PodCIDR

<font color='red'> PodCIDR 与 PodCIDRs 区别？</font>

#### (5) ProviderID

当节点是公有云厂商提供的云主机时，这个属性表示公有云系统中对云主机的唯一标识，格式为：`<ProviderName>://<ProviderSpecificNodeID>`

#### (6) Node 管理

##### 一、node 调度之禁止调度（平滑维护）-cordon、drain、delete

cordon、drain 和 delete 三个命令都会使 node 停止被调度，后期创建的 pod 不会继续被调度到该节点上，但操作的暴力程度却不一样。

**1. cordon 停止调度**（之后不可调度，临时从 K8S 集群隔离）

- 影响最小，只会将 node 标识为 SchedulingDisabled 不可调度状态。
- 之后 K8S 再创建的 pod 资源，不会被调度到该节点。
- 旧有的 pod 不会受到影响，仍正常对外提供服务。
- 禁止调度命令"kubectl cordon node_name"。
- 恢复调度命令"kubectl uncordon node_name"。（恢复到 K8S 集群中，变回可调度状态）

**2. drain 驱逐节点**（先不可调度，然后排干）

- 首先，驱逐 Node 上的 pod 资源到其他节点重新创建。
- 接着，将节点调为 SchedulingDisabled 不可调度状态。
- 禁止调度命令"kubectl drain node_name --force --ignore-daemonsets --delete-local-data"
- 恢复调度命令"kubectl uncordon node_name"。（恢复到 K8S 集群中，变回可调度状态）
- drain 方式是安全驱逐 pod，会等到 pod 容器应用程序**优雅**停止后再删除该 pod。
- drain 驱逐流程：先在 Node 节点删除 pod，然后再在其他 Node 节点创建该 pod。所以为了确保 drain 驱逐 pod 过程中不中断服务（即做到"无感知"地平滑驱逐），必须保证要驱逐的 pod 副本数大于 1，并且采用了<font color="red">"反亲和策略"</font>将这些 pod 调度到不同的 Node 节点上了！也就是说，在"多个 pod 副本+反亲和策略"的场景下，drain 驱逐过程对容器服务是没有影响的。

需要注意：

- 对节点执行维护操作之前（例如：内核升级，硬件维护等），您可以使用 kubectl drain 安全驱逐节点上面所有的 pod。
- drain 安全驱逐方式将会允许 pod 里面的容器遵循指定的 <font color="red">PodDisruptionBudgets </font>执行优雅中止。也就是说，drain 安全驱逐可以做到：优雅地终止 pod 里的容器进程。
- kubectl drain 返回成功表明所有的 pod （除了排除的那些）已经被安全驱逐（遵循期望优雅的中止期，并且没有违反任何应用程序级别的中断预算）。
- 然后，通过对物理机断电或者在云平台上删除节点所在的虚拟机，都能安全的将节点移除。

<font color='red'>如下尚未学习</font>

一般线上 K8S 的 PDB（PodDisruptionBudgets）配置的也是符合 Pod 驱逐的理想情况的，即 maxUnavailable 设置为 0，maxSurge 设置为 1：

```yaml
replicas: 3
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
  type: RollingUpdate
```

默认情况下，kubectl drain 会忽略那些不能杀死的系统类型的 pod。drain 命令中需要添加三个参数：--force、--ignore-daemonsets、--delete-local-data

- --force 当一些 pod 不是经 ReplicationController, ReplicaSet, Job, DaemonSet 或者 StatefulSet 管理的时候就需要用--force 来强制执行 (例如:kube-proxy)
- --ignore-daemonsets 无视 DaemonSet 管理下的 Pod。即--ignore-daemonsets 往往需要指定的,这是因为 deamonset 会忽略 unschedulable 标签(使用 kubectl drain 时会自动给节点打上不可调度标签),因此 deamonset 控制器控制的 pod 被删除后可能马上又在此节点上启动起来,这样就会成为死循环.因此这里忽略 daemonset。
- --delete-local-data 如果有 mount local volumn 的 pod，会强制杀掉该 pod。

<font color='red'>为什么能优雅的驱逐？和先驱逐再新建的顺序有关还是和 PodDisruptionBudgets 策略有关</font>

**3. delete 删除节点**

- 首先，驱逐 Node 节点上的 pod 资源到其他节点重新创建。
- 驱逐流程：先在 Node 节点删除 pod，然后再在其他 Node 节点上创建这些 pod。
- node 节点删除，master 失去对其控制，该节点从 k8s 集群摘除。
- delete 是一种暴力删除 node 的方式。在驱逐 pod 时是强制干掉容器进程，做不到优雅终止 Pod。相比较而言，显然 drain 更安全。

恢复调度（即重新加入到 K8S 集群中）

- delete 删除后，后续如果需重新加入 K8S 集群。则需要重启 node 节点的 kubelet 服务，重启后，基于 node 的自注册功能，该节点才能重新加入到 K8S 集群，并且恢复使用（即恢复可调度的身份）。
- 另外：如果 kubelet 服务重启后，node 节点系统时间跟其他节点不一致，则导致该节点证书会失效！kubelet 注册后，还需要手动 approve 签发 TLS 认证操作了。如下示例：

##### 二、 node 添加（向 API server 添加）

##### 1. 节点的 kubelet 向控制面自注册

- 当 kubelet 标志 `--register-node` 为 true（默认）时，它会尝试向 API 服务注册自己。 这是首选模式，被绝大多数发行版选用。

##### 2. 手动添加 node 对象

```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

**说明：** Kubernetes 会一直保存着非法节点对应的对象，并持续检查该节点是否已经 变得健康。 你，或者某个[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)必需显式地删除该 Node 对象以停止健康检查操作。

Node 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。

#### (7) 节点控制器

节点[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)是 Kubernetes 控制面组件，管理节点的方方面面。

节点控制器在节点的生命周期中扮演多个角色。 第一个是当节点注册时为它分配一个 CIDR 区段（如果启用了 CIDR 分配）。

第二个是保持节点控制器内的节点列表与云服务商所提供的可用机器列表同步。 如果在云环境下运行，只要某节点不健康，节点控制器就会询问云服务是否节点的虚拟机仍可用。 如果不可用，节点控制器会将该节点从它的节点列表删除。

第三个是监控节点的健康状况。 节点控制器是负责：

- 在节点节点不可达的情况下，在 Node 的 `.status` 中更新 `NodeReady` 状况。 在这种情况下，节点控制器将 `NodeReady` 状况更新为 `ConditionUnknown` 。
- 如果节点仍然无法访问：对于不可达节点上的所有 Pod 触发 [API-发起的逐出](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/api-eviction/)。 默认情况下，节点控制器 在将节点标记为 `ConditionUnknown` 后等待 5 分钟 提交第一个驱逐请求。

节点控制器每隔 `--node-monitor-period` 秒检查每个节点的状态。

### 3.2 Namespaces

- 在一个 Kubernetes 集群中可以使用 namespace 创建多个 “虚拟集群”，实现集群间资源的隔离

#### (1) 一些细节点

- 删除一个 namespace 会自动删除所有属于该 namespace 的资源。
- default 和 kube-system 命名空间不可删除；default、kube-system 以及 kube-public 都是默认命名空间
- 用户的普通应用在 defalut 下面
- 与集群管理相关的为整个集群提供服务的应用一般部署在 `kube-system`
- PersistentVolume 是不属于任何 namespace 的，但 PersistentVolumeClaim 是属于某个特定 namespace 的。
- Event 是否属于 namespace 取决于产生 event 的对象。
- v1.7 版本增加了 kube-public 命名空间，该命名空间用来存放公共的信息，一般以 ConfigMap 的形式存放。

<font color="red"> namespaces 不包括 node 和 persistentVolume，那为啥 dashboard 可以实现 node 隔离呢？</font>

### 3.3 Label

- Label 是附着到 object 上（例如 Pod）的键值对。

```json
"labels": {
  "key1" : "value1",
  "key2" : "value2"
}
```

- k8s 将 lables 最终索引和反向索引用来优化查询和 watch，不要在 label 中使用大型、非标识的结构化数据，记录这样的数据应该用 annotation

#### (1) 使用 label 情形

- 可以尝试如下标签

  - `"release" : "stable"`, `"release" : "canary"`

  - `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`

  - `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`

  - `"partition" : "customerA"`, `"partition" : "customerB"`

  - `"track" : "daily"`, `"track" : "weekly"`

  - `"team" : "teamA"`,`"team:" : "teamB"`

#### (2) 语法规则

**Key**

- 不能超过 63 个字符
- 可以使用前缀，前缀使用`/`分隔，前缀需要是 DNS 子域，不得超过 253 个字符，系统中自动化组件创建的 lable 必须指定前缀
- 起始必须是字母（大小写都可以）或数字，中间可以有连字符、下划线和点

**Value**

- 不得超过 63 个字符
- 起始必须是字母（大小写都可以）或数字，中间可以有连字符、下划线和点

#### (3) Selector 选择器

- label 不是唯一的，多个对象可能有相同的 label
- 通过 selector 可以选择一个 obj 集合，对这个集合进行整体的操作

##### 1. 方式一： `equality-based` 基于等值

- 可以使用操作符`=`、`==`、`!=`
  - `=` 和 `==` 是一个意思
- 可使用逗号分隔多个表达式

##### 2. 方式二：`set-based` 基于集合

- 可以使用 `in`、`notion`、`!`操作符
- 可以直接写出某个 label-key，表示直接选中，而不管对应的 value 是何值
- `！`表示没有该 label 的那些 obj

```bash
environment in (production, qa)
# env 等于product或者qa

tier notin (frontend, backend)
# tier 不等于frontend或者backend的资源

partition
# 键为partition

!partition
# 键不为partition的全部
```

##### 3. 资源中示例使用 selector

1. **pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: cuda-test
spec:
	containers:
		- name: cuda-test
			image: "k8s.gcr.io/cuda-vector-add:v0.1"
			resources:
				limits:
					nvidia.com/gpu: 1
		nodeSelector:
			accelerator: nvidia-tesla-p100
```

2. **service 和 ReplicationController**

```json
"selector": {
	"component": "redis",
}
```

```yaml
selector:
component: redis
```

- 等价于 component=redis 或者 component in（redis）

3. **job、Deployment、Replica Set 和 DemonSet 支持基于集合的需求**

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
```

- `matchLabels` 是由 `{key,value}` 对组成的映射
- 有效的运算符包括 `In`、`NotIn`、`Exists` 和 `DoesNotExist`
- 来自 `matchLabels` 和 `matchExpressions` 的所有要求都按逻辑与的关系组合到一起,它们必须都满足才能匹配

##### 4. 通过标签/selector 将 pod 分配给节点

场景：例如，确保 Pod 最终落在连接了 SSD 的机器上，或者将来自两个不同的服务 且有大量通信的 Pods 放置在同一个可用区。

- 一个简单的 nginx 配置文件

```yaml
apiVerions: v1
kind: Pod
metadata:
	name: nginx-hxc
	labels:
		env: test
spec:
	containers:
		- name: nginx
			image: nginx
			imagePullPolicy: IfNotPresent
	nodeSelector:
		test/role: houWorker
```

#### (4) 亲和性与反亲和性

**相比于 selector 优点**

- 语言更具表现力，（不局限于只能使用完全匹配且 AND 连接的规则）
- 可以设置“软需求”，即如果实在无法满足要求，仍然是可调度的
- 可以使用节点上的 pod 的标签来约束，而不仅仅是节点的标签

##### 一、节点亲和性

- `requiredDuringSchedulingIgnoredDuringExecution` “硬需求”

  必须满足

- `preferredDuringScheduingIgnoredDuringExecution` “软需求”

  尽量满足

- `IgnoredDuringExecution` 意味着，在 pod 运行时，如果标签不再满足要求也不会被驱逐

**节点亲和性 affinity 部署示例**

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: with-node-affinity
spec:
	containers:
  - name: with-node-affinity
  	image: k8s.gcr.io/pause:2.0
  	imagePullPolicy: IfNotPresent
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
        - matchExpressions:
        	- key: test.com/role
        		operator: In
        		values:
        		- hxcWorker
        		- hxcWorker2
			preferredDuringSchedulingIgnoredDuringExecution:
			- weight: 1
				preference:
					matchExpressions:
					- key: another/role
						operator: In
						values:
						- anotherWorker
```

- 亲和性支持的操作符 `In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt`
- 反亲和性操作符 `NotIn`、`DoesNotExist`
- 如果同时指定`nodeSelector`和`nodeAffinity`，则两者都要满足，才能将 pod 调度到节点上
- 一个`nodeSelectorTerms`中的任意一个`matchExpressions`满足就可以调度

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms: # array
      - matchExpressions:
          - key: test.com/role
            operator: In
            values:
              - hxcWorker
              - hxcWorker2
          - key: match
            operator: In
            values:
              - yep
      - matchExpressions:
          - key: match2
            operator: In
            values:
              - hxcWorker
              - hxcWorker2
# 两个matchExpressions满足一个即可
```

- 一个`matchExpression`内的全部规则都匹配才能调度

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms: # array
      - matchExpressions:
          - key: test.com/role
            operator: In
            values:
              - hxcWorker
              - hxcWorker2
          - key: match
            operator: In
            values:
              - yep
# test.com/role和match两条标签均要满足才能调度
```

- 亲和性也是只在节点调度时有效，即如果运行时，标签修改，也不会驱逐节点
- `preferredDuringSchedulingIgnoredDuringExecution`中的`weight`字段范围时 1-100，最终通过总分与其他节点 pk

##### 二、Pod 间亲和性/反亲和性（约束 pod 标签而不是节点标签）

- pod 间亲和性需要大量的处理，会影响集群调度的速度，如果集群规模过大，不建议使用

```yaml
# 示例
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity: # 关键结构一：podAffinity
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: security
                operator: In
                values:
                  - S1
          topologyKey: topology.kubernetes.io/zone
    podAntiAffinity: # 关键结构二：podAntiAffinity
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: security
                  operator: In
                  values:
                    - S2
            topologyKey: topology.kubernetes.io/zone
  containers:
    - name: with-pod-affinity
      image: k8s.gcr.io/pause:2.0
```

**实际应用：web 服务区与缓存 redis 尽可能放置在同一节点上**

## 四、控制器

### 4.1 Deployment

- deployment 会去创建对应的 rs，rs 再去创建对应的 pod
- dp 支持滚动升级和回滚应用（通过控制 rs 的版本） 比如滚动更新会创建新的 rs，但是旧的 rs 不删除，这样就可以实现回滚
- dp 支持暂停和继续（通过控制 rs 存在的数量？？？）

### 4.2 StateFulSet

有状态服务不太好处理，docker 是通过持久化存储卷的方式来实现，k8s 提供了 statefule 来支持

但是对于 mysql，可能现在的支持还不是特别的完善

- 通过稳定的持久化存储，pod 重新调度后能够访问到相同的持久化数据，基于 PVC 来实现
- 稳定的网络标识：PodName 和 HostName 不变，基于 headless service(即没有 cluster ip 的 service)来实现
- 有序部署，pod 的部署是有前后顺序的，基于 init C 来实现
- 有序收缩，有序删除

### 4.3 DaemonSet<font color="red">(还不是特别理解)</font>

DaemonSet 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用 DaemonSet 的一些典型用法：

- 运行集群存储 daemon，例如在每个 Node 上运行 `glusterd`、`ceph`。
- 在每个 Node 上运行日志收集 daemon，例如`fluentd`、`logstash`。
- 在每个 Node 上运行监控 daemon，例如 [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)、`collectd`、Datadog 代理、New Relic 代理，或 Ganglia `gmond`。

一个简单的用法是，在所有的 Node 上都存在一个 DaemonSet，将被作为每种类型的 daemon 使用。 一个稍微复杂的用法可能是，对单独的每种类型的 daemon 使用多个 DaemonSet，但具有不同的标志，和/或对不同硬件类型具有不同的内存、CPU 要求。

### 4.4 ReplicationController （已经是过去时了）， ReplicaSet

- RC 就是最基本，维持 pod 副本数不多也不少
- RS 与 RC 没有什么不同，支持集合式的 selector
- RS 具有扩容、缩容功能

```shell
kubectl explain rs
```

### 4.5 Job、 CronJob

处理脚本执行的问题

- job 如果运行脚本没有 0 成功退出，会再次重新运行

- cronjon 只是在 job 的基础上实现周期性运行（通过在特定的时间循环创建 job 来实现）

### 4.6 Horizontal Pod Autoscaling （水平自动扩展）

- 可以理解为不是一个控制器，而是一个控制器的附属品
- 创建了一个 dp 之后，可以再创建一个 hpa 来管理控制 dp
- 可以通过一些资源指标（CPU，mem）等来实现 pod 缩放

## 五、服务发现与路由

### 5.1 service

**每个 svc 可以理解为一个微服务**

#### (1) svc 负载均衡

- 只能实现 4 层负载均衡，没有 7 层负载均衡能力

#### (2) svc 类型

- **clusterIP** ： 自动分配一个 cluster 内部（节点内部）可以访问的 IP，即节点内其他 pod 可以通过这个 ip 访问到 svc 即访问到 svc 对应的 pod
- **NodePort** ： 在 clusterIp 的基础上，分配了一个端口，可以通过 IP：port 方式来访问服务，这种方式就将服务暴露到外部去了，一般端口是会做一个映射的 80->30001 这种
- **LoadBalancer** ：在 nodeport 的基础上，借助 cloud provider 创建一个外部负载均衡器，将请求转发到 nodeIP：nodeport，相当于 svc 可能会在好几个 node 节点上都创建多个 pod，那么可以在这些 node 之前加一个负载均衡器来实现对于 nodeIP：port 这种方式的 svc 的访问，均衡器可以是自己整的 nginx，这里则是指云服务提供商提供的服务
- **ExternalName** ： 只是为了在集群内部定义一个统一的对外面的某个访问的 svc，因为可能集群内部多个 pod 都会要访问这个外部服务，但是如果外部 IP 变了的话，就要改动很大，而使用 ExternalName 这种 svc 的话，则只用改这个 svc 中的配置

#### (3) svc 原理

<img src="/img/image-20211126173238085.png" alt="image-20211126173238085" style="zoom:50%;" />

- client 访问 svc，转发到对应的 pod 是通过 iptables 来实现的
- iptables 规则的生成是通过 kube-proxy（服务发现）来实现的

## 六、 身份与权限控制

### 6.1 kubeconfig

- kubeconfig 是 kubernetes 集群中各个组件(api-server 客户端)连入 api-server 的时候 , 使用的认证格式的客户端配置文件

- 查看 kubeconfig 配置文件

`kubectl config view`

- kubeconfig 证书 和 token 两种认证方式是 K8S 中最简单通用的两种认证方式

##### 生成 kubeconfig 的配置步骤

1、定义变量
export KUBE_APISERVER="https://172.20.0.2:6443"

2、设置集群参数 kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true --server=\${KUBE_APISERVER} #可以指定路径 kubeconfig=/root/config.conf 说明：集群参数主要设置了所需要访问的集群的信息。使用 set-cluster 设置了需要访问的集群，如上为 kubernetes；--certificate-authority 设置了该集群的公钥；--embed-certs 为 true 表示将 --certificate-authority 证书写入到 kubeconfig 中；--server 则表示该集群的 kube-apiserver 地址。

3、设置客户端认证参数 kubectl config set-credentials admin --client-certificate=/etc/kubernetes/ssl/admin.pem --embed-certs=true --client-key=/etc/kubernetes/ssl/admin-key.pem #可以指定路径 kubeconfig=/root/config.conf 说明：用户参数主要设置用户的相关信息，主要是用户证书。如上的用户名为 admin，证书为：/etc/kubernetes/ssl/admin.pem，私钥为：/etc/kubernetes/ssl/admin-key.pem。注意客户端的证书首先要经过集群 CA 的签署，否则不会被集群认可。此处使用的是 ca 认证方式，也可以使用 token 认证，如 kubelet 的 TLS Boostrap 机制下的 bootstrapping 使用的就是 token 认证方式。

4、设置上下文参数
kubectl config set-context kubernetes --cluster=kubernetes --user=admin #可以指定路径 kubeconfig=/root/config.conf 说明：上下文参数将
**集群参数**和**用户参数**关联起来。如上面的上下文名称为 kubenetes，集群为 kubenetes，用户为 admin，表示使用 admin 的用户凭证来访问 kubenetes 集群的 default 命名空间，也可以增加 --namspace 来指定访问的命名空间。

5、设置默认上下文 kubectl config use-context kubernetes #可以指定路径 kubeconfig=/root/config.conf

说明：最后使用`kubectl config use-context kubernetes` 来使用名为 kubenetes 的环境项来作为配置。如果配置了多个环境项，可通过切换不同的环境项名字来访问到不同的集群环境。

#### 完整的 k8s 的认证方式

https://zhuanlan.zhihu.com/p/97797321

## 七、 网络

## 八、 存储

### 8.2 ConfigMap

#### (1) 是什么？

- configMap 就是 k8s 中的配置，本质是键值对

#### (2) 为什么用 configMap ? 不是 yml 吗？有哪些地方需要用到配置？

#### (3) 方向（configMap 主要考虑创建和使用两部分）

- 创建
- 使用

- 配置文件（configMap、secret 都存储在 etcd 中了）
- configMap 可以用来保存单个属性，也可以保存配置文件

#### (4) configMap 作用

1. 设置环境变量的值
2. 设置命令行参数
3. 在数据卷中创建 config 文件

#### (5) 与 secret 的区别：

- configMap 是不需要加密的配置，其他基本相同

#### (6) 其他资料

- **配置中心**的概念，比如携程的开源分布式配置中心 Apollo

## 九、 集群扩展

## 十、配置

### 9.1 apiVersion

- Deployment 可用 apps/v1

  ```apache
  1.6版本之前 apiVsersion：extensions/v1beta1

  1.6版本到1.9版本之间：apps/v1beta1

  1.9版本之后:apps/v1
  ```

- pods 可以直接是 v1

### 9.2 spec

- 对于 Deployment，spec 下的 selector 字段是 required 的

meta 理解为类型属性的描述 metav1.TypeMeta -> kind/apiVersion

spce：定义 API 对象类型私有属性。 -> spec. (也是这个字段使得不同 API 对象得以区分)

Status : 描述对象的状态

