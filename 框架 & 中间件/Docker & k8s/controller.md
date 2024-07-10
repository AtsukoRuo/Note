# Controller

[TOC]



## Deployment

Pod 没有自愈能力，不能扩缩容，也不支持方便的升级和回滚。而 Deployment 可以。因此，建议绝大多数情况下采用 Deployment 来部署 Pod。Deployment 在底层利用了另一种名为 ReplicaSet 的对象。并不建议直接操作 ReplicaSet。

![image-20240616113047893](./assets/image-20240616113047893.png)

ReplicaSet 的底层调谐循环确保当前状态与期望状态保持一致，以提供自愈与扩缩容能力。下面我们来看看如何在零宕机的情况下实现了一种平滑的滚动升级。

现在假设某人遇到了一个 Bug，并且需要部署一个新的镜像来完成修复。因此，他修改了同一个 Deployment 的 YAML 文件，将镜像版本更新，并重新 POST 到 API Server。为了到达期望状态，Kubernetes 基于新镜像的 Pod 创建了一个新的 ReplicaSet。此时就有两个 ReplicaSet 了：一个是包含基于旧版镜像的 Pod，一个是新版本的 Pod。每次 Kubernetes 增加新 ReplicaSet（新版镜像）中的 Pod 数量的时候，都会相应地减少旧 ReplicaSet（旧版镜像）中的 Pod 数量。这一切只需更新清单文件！

着重需要说明的是，旧版的 ReplicaSet 仍然有完整的配置信息，包括旧版的镜像，这对于回滚功能来说很重要。

![image-20240616121043866](./assets/image-20240616121043866.png)



下面，我们来创建一个 Deployment。首先创建 manifest 文件

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata: 			# Deployment 自己的 metadata 
  name: hello-deploy
spec:
  replicas: 10		# 需要多少个副本
  selector:			# 表明 Deployment 所要管理的 Pod 必须具备的标签
    matchLabels:
      app: hello-world
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10			# 每个 Pod 的更新操作要间隔 10s
  strategy:
    type: RollingUpdate	# 如何执行更新操作
    rollingUpdate:
      # 下面两个参数表明，在滚动更新的过程中，最多只能同时更新两个 Pod
      maxUnavailable: 1		# 不允许出现比期望状态指定的 Pod 数量少超过一个的情况，即在更新过程中，Pod 数量不能超过 11 个
      maxSurge: 1			# 不允许出现比期望状态指定的 Pod 数量多超过一个的情况，即即在更新过程中，Pod 数量不能少于 9 个
  template:		# Pod 模板
    metadata:
      labels:
        app: hello-world
    spec:
      containers:	
      - name: hello-pod		# 容器名
        image: nigelpoulton/k8sbook:1.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: 128Mi
            cpu: 0.1
~~~

然后 POST 到 API Server 中

~~~shell
$ kubectl apply -f deploy.yml
~~~

通过 kubectl get 或者 kubectl describe 命令来查看 Deployment 对象：

~~~shell
$ kubectl get deploy hello-deploy
$ kubectl describe deploy hello-deploy
~~~

现在，我们提供一个 Service 对象，对 Deployment 中的 Pod 对象提供一个稳定的、向外暴露的 IP 地址。

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  labels:
    app: hello-world
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
    protocol: TCP
  selector:
    app: hello-world
~~~

~~~shell
$ kubectl apply -f svc.yml
~~~

现在 Service 已经部署好了，可以通过以下任一种方式来访问该应用：

- 在集群内部，通过 DNS 名称 hello-svc （Service 的名称）和端口 8080 访问
- 在集群外部，通过集群任意一个节点的 IP 和端口号 30001 访问



现在只要修改 manifest 文件中的 spec.containers.image，然后 POST 即可实现更新操作。执行 kubectl apply 来 POST manifest 文件时，要附带 --record 参数，这对回滚操作很重要。

~~~shell
$ kubectl apply -f deploy.yml --record
~~~

可以执行 `kubectl rollout status`来查看更新过程：

~~~shell
$ kubectl rollout status deployment hello-deploy
Waiting for rollout to finish: 4 out of 10 new replicas... 
Waiting for rollout to finish: 4 out of 10 new replicas... 
Waiting for rollout to finish: 5 out of 10 new replicas...
~~~

由于在升级 Deployment 时使用了 --record 参数。因此，在执行 kubectl rollout history 命令时，可以显示 Deployment 的两个版本：

~~~shell
$ kubectl rollout history deployment hello-deploy
deployment.apps/hello-deploy
REVISION CHANGE-CAUSE 
1		<none> 
2		kubectl apply --filename=deploy.yml --record=true
~~~

下面通过 kubectl rollout 命令来回滚到版本 1：

~~~shell
$ kubectl rollout undo deployment hello-deploy --to-revision=1
~~~



## Service

Pod 的 IP 地址是不可靠的：

- 在某个 Pod 失效之后，它会被一个拥有新的 IP 的 Pod 代替
- Deployment 扩容也会引入拥有新 IP 的 Pod；而缩容则会删除 Pod。

每一个 Service 都拥有固定的 IP 地址、固定的 DNS 名称，以及固定的端口。此外，Service 还可以利用 Label ，来动态选择将流量转发至哪些 Pod。

Service 与 Pod 之间是通过 Label 和 Label 筛选器（selector）松耦合在一起的。Deployment 与 Pod 也是这样绑定在一起的，但 Deployment 只会考虑由自己创建的 Pod。所有匹配的 Pod 必须拥有 Service Label 筛选器中定义的所有 Label。下面我们通过三个例子来认识这一点

![image-20240616122554688](./assets/image-20240616122554688.png)

![image-20240616122622652](./assets/image-20240616122622652.png)

![image-20240616122629294](./assets/image-20240616122629294.png)



下面是 Service 如何工作的基本介绍

1. 每一个 Service 在被创建的时候，都会得到一个关联的 Endpoint 对象。整个 Endpoint 对象其实就是一个**动态**列表，其中包含匹配该 Service Label 筛选器的健康 Pod 的 IP。 
2. 当要通过 Service 将流量转发到 Pod 时，首先在集群内部的 DNS 中查询 Service 的 IP。当节点将流量打到 ClusterIP 时，会被内核捕获拦截（具体来说是每个节点都有的 kube-proxy 系统服务），然后将 IP 改写为某个健康 Pod 的 IP 地址。
3. Kubernetes 原生应用是可以直接查询 Endpoint API，而无须查找 DNS 和使用 Service IP 的。



Kubernetes 支持几种不同类型的 Service：

- ClusterIP Service
- NodePort Service
- LoadBalancer Service

 `ClusterIP Service`（默认）：拥有固定的 IP 地址和端口号，并且仅能够从集群内部访问得到。在创建 ClusterIP Service 时，ClusterIP 与对应的 Service 名称（`metadata.name`）会被动地注册注册在集群内部的 DNS 服务中（下面会解释的）。所以， Pod（应用的微服务）只需知道 Service 的名称，就能够解析到对应的 ClusterIP，然后将流量打到 ClusterIP 上，Service 就自动帮我们转发流量到相应的 Pod 上。

`NodePort Service`：在 ClusterIP Service 的基础上，增加了另一个端口 NodePort，从而可以从集群外部访问到 Service。以下的例子表示一个 NodePort Service

~~~json
Name:magic-sandbox
ClusterIP:172.12.5.17
port:8080
NodePort:30050
~~~

- 在集群内部，可以通过`(ClusterIP、port)`来直接访问这个服务。
- 在集群外部，可以发送请求到集群中的任何一个节点上的 `NodePort` 端口来访问该服务。

下面给出一个访问 NodePort Service 的例子

![image-20240702021350813](./assets/image-20240702021350813.png)

1. 来自一个外部客户端的请求到达 Node2 的 30050 端口
2. 请求被转发至 Service 对象
3. 从 Endpoint 中获取健康的 Pod
4. 请求被转发至 Node1 上的 Pod1



`LoadBalancer Service` 基于 NodePort，并且集成了基于云的负载均衡器



Kubernetes 通过以下方式来实现**服务发现**（Service discovery）

- DNS（推荐）
- 环境变量（绝对不推荐）

基于 DNS 的服务发现需要 DNS 集群插件（cluster-add-on），几乎所有的 Service 都使用到了它。Kubelet 为每一个容器都注入了该 DNS（通过/etc/resolv.conf），这也就意味着任何 Pod 都可以通过连接到 `kube-dns` 服务来进行 DNS 解析。这个 DNS 插件（kube-dns）会持续监测 API Server。一旦发现有新创建的 Service 对象，就会创建相应的 DNS 记录。因此，应用和 Service 无须主动执行服务注册。

关于环境变量方式的最大问题在于，环境变量只有在 Pod 最初创建的时候才会被注入。这就意味着，Pod 在创建之后是并不知道新 Service 的。



Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为**命名空间**，它们在逻辑上彼此隔离。这可以作为在多个团队之间访问控制和资源限额的一种手段。不过，它不能作为流量隔离的手段来使用。这里仅仅介绍如果创建

Kubernetes 启动时会创建四个初始名字空间：

1. **default**：如果一个对象并没有指定命名空间，那么它就会放入到 default 中
2. **kube-node-lease**
3. **kube-public**
4. **kube-system**

当你创建一个 Service 时， Kubernetes 会创建一个相应的 DNS 条目，该条目的形式是`<object-name>.<namespace>.svc.cluster.local`。这也被称为 Service 对象的**全限定域名（FQDN）**。在下面例子中，prod 命名空间中的 Pod 可以使用短名称（比如 ent 和 voy）来访问本命名空间内部的 Service。而如果需要连接其他命名空间中的 Service，则需要使用 FQDN，比如 `ent.dev.svc.cluster.local`作为域名。

![image-20240702101058933](./assets/image-20240702101058933.png)

下面给出一个命名空间的示例：

~~~~yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
~~~~

Deployment 对象的命名空间绑定

~~~yaml
apiVersion: v1
kind: Deployment
metadata:
  name: enterprise
  labels:
    app: enterprise
  namespace: prod
spec:
~~~

单独 Pod 对象（不放在 Controller 中）的命名空间绑定

~~~yaml
apiVersion: v1
kind: Pod
metadata:
   name: jump
   namespace: dev
~~~



Service 可以用在简单的灰度发布上，我们只需修改该 Label Selector 即可完成流量分配。

![image-20240702023018632](./assets/image-20240702023018632.png)

![image-20240702023028336](./assets/image-20240702023028336.png)

## StatefulSet 

Deployment 和 StatefulSet 都支持自愈、自动扩缩容、滚动更新等特性。但是 StatefulSet  能够确保即使发生故障、扩缩容，调度等操作之后， Pod 名字、DNS 主机名、卷的绑定之间都是保持不变。这 3 个属性构成了 Pod 的状态。

举个简单的例子，由 StatefulSet 管理的 Pod，在发生故障后会被新的 Pod 代替，不过依然保持相同的名字、相同的 DNS 主机名和相同的卷。即使新的 Pod 在另一个节点上启动，亦是如此。

下面就是一个典型的 StatefulSet 定义：

~~~yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: tkb-sts 
spec: 
	selector:
		matchLabels:
			app: mongo 
	ServiceName: "tkb-sts" 
	replicas: 3 
	template: 
		metadata:
			labels:
				app: mongo
	spec:
		containers:
		- name: ctr-mongo 
		image: mongo:latest
~~~

StatefulSet 的 Pod 名字遵循`<StatefulSetName>-<Integer>`的规则。其中 Integer 是一个从零开始的索引号



关于 StatefulSet 的另一个基本特性就是，对 Pod 的启动和停止是受控和有序的。就是说在扩容时，StatefulSet 等前一个 Pod 达到运行且就绪状态之后，才开始创建下一个 Pod。

![image-20240702224742674](./assets/image-20240702224742674.png)

当缩容时，控制器会首先终止拥有最高索引号的 Pod，等待其被完全删除之后，再继续删除下一个拥有最高索引号的 Pod。还可以借助 `terminationGracePeriodSeconds` 这样的参数来调整间隔时间，以控制缩容速度。但请注意，删除一个 StatefulSet 并不是按序依次终止所有 Pod 的。

当一个 StatefulSet Pod 被创建时，所需的卷也会被创建，每一个 Pod 和卷（PVC）都是通过名字建立正确的绑定关系的。

![image-20240702225259829](./assets/image-20240702225259829.png)

如果 StatefulSet 的一个 Pod 在缩容操作中被删除，则当 StatefulSet 被再次扩容时，新增的 Pod 会通过名字的匹配，继续连接到之前已存在的卷上。当进行 Pod 的替换时，只需要保持名字不变，就可以连接同一个卷。





headless Service，就是一个将 spec.clusterIP 设置为 None 的 Service 对象。当这个 headless Service 被设置为 StatefulSet 的 spec.ServiceName 时，它就成为了 StatefulSet 的 governing Service。在二者如此关联之后，Service 会为所匹配的每个 Pod 副本创建 DNS SRV 记录。其他 Pod 可以通过对 headless Service 发起 DNS 查询来获取 StatefulSet 的信息。下面给出一个例子

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-prod
spec: 
  clusterIP: None		# 一个 headless Service
  selector:
  app: mongo
  env: prod
~~~

~~~yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-mongo
spec: 
  ServiceName: mongo-prod			# 通过名字来指定与这个 StatefulSet 关联的 Headless Service
~~~

Headless Service 与常见的 Service 对比：

- **正常的 Service**：适用于无状态应用，流量会通过 Service 的负载均衡策略分发到多个 Pod 上。
- **Headless Service**：适用于有状态应用，流量可以直接路由到特定的 Pod，适用于需要知道具体 Pod 地址的应用场景。

通过 `<object-name>.<Service-name>.<namespace>.svc.cluster.local`  域名来查询特定 Pod 的地址



下面给出一个例子：

~~~yaml
# 部署 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: flash
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-ssd
  
# 部署 governing headless Service
apiVersion: v1
kind: Service
metadata:
  name: dullahan
  labels:
    app: web
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: web
    
# 部署 StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tkb-sts  # StatefulSet的名字
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  serviceName: "dullahan"	# 通过名字来指定与这个 StatefulSet 关联的 Headless Service
  
  # Pod 模板 
  template:
    # 略
  volumeClaimTemplates:
  # 每个 Pod 都关联两个 PVC
  - metadata:
      name: webroot
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "flash"		# 通过名字指定 StorageClass
      resources:
        requests:
          storage: 1Gi
  - metadata:
      name: logs
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 500Mi
~~~

`volumeClaimTemplate` 用于在每次创建一个新的 Pod 副本的时候，自动创建一个 PVC，并且为 PVC 命名，以便实现与 Pod 的准确关联。

