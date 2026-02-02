# 5. StatefulSet 有状态控制器

## 5.1 有无状态服务概念

### 5.1.1 有状态服务

StatefulSet是有状态的集合，管理有状态的服务，**它所管理的Pod的名称和持久化数据不能随意变化**。每一个Pod都有自己独有的数据持久化存储目录。比如MySQL主从、Redis集群（主从、哨兵）等

### 5.1.2 无状态服务

RS、Deployment、DaemonSet都是管理无状态的服务，它们所管理的**Pod的IP、名字，启停顺序等都是随机的**。个体对整体无影响，通常都是所有pod都是共用一个数据卷的。

## 5.2 StatefulSet 的特点

- **唯一性：**对于包含 N 个副本的 StatefulSet，每个 pod 会被分配一个 [0-(N-1)]范围内的唯一序号。
- **顺序性：**StatefulSet 中 pod 的启动、更新、销毁默认都是按顺序进行的。
- **稳定、唯一的网络标识符：**pod 的主机名、DNS 地址不会随着 pod 被重新调度而发生变化（必须配合无头服务）。
- **稳定的持久化存储：**当 pod 被重新调度后，仍然能挂载原有的 PersistentVolume，保证了数据的完整性和一致性。

## 5.3 StatefulSet 组成部分

- **Headless Service**：定义稳定、唯一的网络标识符，生成可解析的DNS记录
- **volumeClaimTemplates**：存储卷申请模板，自动创建pvc，且pvc由存储类供应
- 定义**StatefulSet**来创建pod

## 5.4 Headless Service

==Headless Service，即无头服务，与service的区别就是它没有Cluster IP，解析无头服务的域名时将返回该无头服务对应的全部Pod的Endpoint列表（Pod的IP），而解析普通的svc时，只返回svc的ClusterIP，不返回Pod的IP。==

不管是普通的svc还是无头服务，它们在创建后都会通过集群中的CoreDNS组件自动的生成一个域名，它们的格式都是：

```bash
<Service名称>.<命名空间>.svc.cluster.local
<无头服务名称>.<命名空间>.svc.cluster.local
```

创建一个无头服务再关联一个statefulset，然后查看svc域名的解析结果

```bash
[root@master 1.3]# cat 1.headless.yaml 2.sts.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: headless
spec:
  clusterIP: None  # None就是无头服务，没有clusterIP
  selector:
    app: myapp
  type: ClusterIP
status:
  loadBalancer: {}
---
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysts
spec:
  serviceName: headless # 指定无头服务
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: web
        image: crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/nginx:v1
[root@master 1.3]# kubectl apply -f 1.headless.yaml 2.sts.yaml 
```

使用`kubectl run --rm -it test --image=crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/alpine:3.20 -- sh`创建一个Pod来验证一下无头服务是否能解析到Pod的IP（因为是在集群中添加的DNS记录，所以在宿主机上是不能解析的）

![image-20260103172503287](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103172503287.png)

也可以在宿主机上执行host命令，后面添加DNS服务器，来解析该svc的域名

![image-20260103173554987](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103173554987.png)

## 5.5 Pod 的固定域名格式

==StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod副本创建了一个DNS域名，这个域名的格式为：==

```bash
<Pod名字>.<无头服务名称>.<命名空间>.svc.cluster.local
```

![image-20260103174722772](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103174722772.png)

通过以上的 Headless Service 操作可以看出，无头服务是和 StatefulSet 紧密相连的，无头服务给 StatefulSet 提供了唯一的网络标识符，不会随着 Pod 被重新调度而发生变化。

## 5.6 创建和删除 StatefulSet

### 5.6.1 创建

先创建一个资源清单文件，StatefulSet的资源清单文件和Deployment的清单差不多，最大的区别就是多了一个可选的`.sepc.serviceName`字段

```bash
[root@master 1.3]# cat 3.mysts.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: headless
spec:
  clusterIP: None  # None就是无头服务，没有clusterIP
  selector:
    app: myapp
  type: ClusterIP
status:
  loadBalancer: {}
---
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysts
spec:
  serviceName: headless # 指定无头服务
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: web
        image: crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/nginx:v1
```

StatefulSet创建Pod的时候，是从0到副本数减1的序号创建的，==只有上一个Pod的状态为Running且Ready了，才开始创建下一个Pod，如果中途有某个Pod没有起来，那么它会一直等待，不会往后创建了，除非查找原因重建sts。==

![image-20260103210635894](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103210635894.png)

### 5.6.2 删除

![image-20260103212957914](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103212957914.png)

## 5.7 StatefulSet 扩缩容

和 Deployment 类似，可以通过`kubectl edit sts <sts名称>`更新 replicas 字段扩容/缩容 StatefulSet，也可以使用 `kubectl scale` 和 `kubectl patch` 来扩容/缩容 StatefulSet。

### 5.7.1 扩容

![image-20260103213857921](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103213857921.png)

### 5.7.2 缩容

![image-20260103214107888](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103214107888.png)

## 5.8 StatefulSet 更新策略

- **RollingUpdate** 策略 （滚动更新，默认），会自动更新 StatefulSet 中所有的 Pod，采用与序号索引相反（倒序）的顺序进行滚动更新。
- **OnDelete** 策略（手动删除后重建，1.7 之前的版本的默认策略），StatefulSet 控制器不会自动更新 Pod，必须手动删除 Pod 才能使控制器创建新的 Pod，可以通过`statefulSet.spec.updateStrategy.type`字段来修改策略。

### 5.8.1 RollingUpdate 策略（默认）

更新镜像来触发滚动更新，只要更改`.spec.template`字段下的内容就会触发更新

```bash
[root@master 1.3]# kubectl set image sts mysts web=crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/nginx:v2
```

![image-20260103215455529](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103215455529.png)

在滚动更新的时候，==它是从最大的序号先前开始更新==，先删除最大的序号的Pod，然后再创建同序号的Pod（必须Running且Ready），然后以此类推，直到全部Pod都更新完。

![image-20260103215706323](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103215706323.png)

==那么如果滚动更新的时候，某一个Pod没有起来，那么它会一直等待该Pod，不会往后进行更新，这个时候可以查找原因进行修改，然后把起不来的Pod删除，让他重新更新该Pod==

### 5.8.2 OnDelete 策略

使用`kubectl edit sts mysts`命令来修改更新的策略，修改为OnDelete

![image-20260103223758712](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103223758712.png)

更新Pod的镜像为nginx:v3版本

```bash
[root@master 1.3]# kubectl set image sts mysts web=crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/nginx:v3
```

![image-20260103224905729](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103224905729.png)

这个时候如果想要更新其他的 Pod ，和以上的方式一样，删除即可完成更新。

## 5.9 StatefulSet 分段更新

可以通过`statefulSet.spec.updateStrategy.rollingUpdate.partition`字段来配置分段更新的规格（要把更新策略改为默认的 rollingUpdate），默认该字段等于0（代表只有序号大于等于 0 的才会更新），可以使用 patch 或 edit 直接对 StatefulSet 进行设置

这里通过`kubectl edit sts mysts`把该字段的值修改为1，也就是说只有序号大于等于1的才会被更新

![image-20260103230347273](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103230347273.png)

更改Pod的镜像来触发更新

![image-20260103231204515](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103231204515.png)

如果更新后都没有问题的话，可以再把 partition 字段的值改为0，实现全部 Pod 的更新，类似于灰度/金丝雀发布。

## 5.10 StatefulSet 版本回滚

StatefulSet 的更新和回滚与 Deployment 类似，唯一的区别是历史版本在 ControllerRevision 中保存的，Deployment是在 RS 中保存的。

```bash
# 查看版本历史
[root@master 1.3]# kubectl rollout history sts mysts
statefulset.apps/mysts 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
4         <none>

# 查看指定的历史版本的详细信息，该版本是一个nginx:v1版
[root@master 1.3]# kubectl rollout history sts mysts --revision=3
statefulset.apps/mysts with revision #3
Pod Template:
  Labels:       app=myapp
  Containers:
   web:
    Image:      crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/nginx:v1
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

# 回滚到指定的版本
[root@master 1.3]# kubectl rollout undo sts mysts --to-revision=3

# 这个时候Pod是按照倒序来依次的回滚的，如有起不来的，会一直等待
[root@master 1.3]# kubectl get pods -owide
NAME      READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
mysts-0   1/1     Running   0          23m   172.16.196.148   node01   <none>           <none>
mysts-1   1/1     Running   0          14s   172.16.140.77    node02   <none>           <none>
mysts-2   1/1     Running   0          47s   172.16.196.153   node01   <none>           <none>
[root@master 1.3]# curl 172.16.140.77
wo shi v1
```

## 5.11 StatefulSet 并发 Pod 管理 

StatefulSet 可以通过`.spec.podManagementPolicy` 字段配置 Pod 的管理策略，目前支持如下的两种方式： 

- **OrderdReady** **（默认方式）**：有序管理，Pod 创建和更新按照正序和倒序进行操作，删除是同时删除的
- **Parallel**：并发管理，Pod 创建和删除同时并行启动和删除（更新不是并发的）。

这里默认的 OrderReady 就不演示了，因为以上的案例都是按照默认的有序方式来操作的，相信大家都已经琢磨明白了。

**Parallel 方式示例**

```bash
[root@master 1.3]# cat 3.mysts.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: headless
spec:
  clusterIP: None  # None就是无头服务，没有clusterIP
  selector:
    app: myapp
  type: ClusterIP
status:
  loadBalancer: {}
---
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysts
spec:
  podManagementPolicy: Parallel # 修改Pod管理方式
  serviceName: headless # 指定无头服务
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: web
        image: crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/nginx:v1
```

创建 StatefulSet 查看 Pod 是否是同时创建的，即使 Pod 是同时创建的没有按顺序创建，那么这些 Pod 的名称还是是按顺序命名的

![image-20260103234110053](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103234110053.png)

删除同理

![image-20260103234538715](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260103234538715.png)

## 5.12 部署 Eureka 集群

可以通过以下的资源清单来创建 Eureka 集群，清单中的 env 下的 `EUREKA_SERVER_ADDRESS` 变量是指定哪些实例来创建 Eureka 集群，所以它的 value 值就是每个 Pod 的唯一网络标识符（域名）加端口和路径

==生产环境中一定要加 Pod 的健康检查==

```bash
[root@master 1.3]# cat 4.eureka_sts.yaml 
apiVersion: v1
kind: Service
metadata:
  name: eureka-headless
spec:
  clusterIP: None  # None就是无头服务，没有clusterIP
  selector:
    app: eureka
  type: ClusterIP
status:
  loadBalancer: {}
---
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: eureka
spec:
  serviceName: eureka-headless # 指定无头服务
  replicas: 3
  selector:
    matchLabels:
      app: eureka
  template:
    metadata:
      labels:
        app: eureka
    spec:
      containers:
      - name: eureka
        image: crpi-j2guy08w46nav7rf.cn-hongkong.personal.cr.aliyuncs.com/xiao_he/eureka:demo
        ports: 
        - containerPort: 8761
          name: eureka
          protocol: TCP
        env: 
        - name:  SPRING_PROFILES_ACTIVE
          value: k8s
        - name: SERVER_PORT
          value: '8761'
        - name: EUREKA_SERVER_ADDRESS
          # 这里要是sts管理的Pod的域名，注意不要写错
          value: http://eureka-0.eureka-headless:8761/eureka/,http://eureka-1.eureka-headless:8761/eureka/,http://eureka-2.eureka-headless:8761/eureka/
        startupProbe: 
          tcpSocket:           # 四种检查方式，每个探针只能使用一个
            port: 8761           # 必填
          initialDelaySeconds: 20   # 容器启动后，多少秒开始进行探测
          timeoutSeconds: 2    # 探测超时的秒数，默认是1秒
          periodSeconds:  5    # 执行探测的频率，默认10秒，最小可配置1秒
          successThreshold: 1  # 检测成功1次表示就绪，默认1次
          failureThreshold: 2  # 检测失败3次表示未就绪，默认3次，最小可配置1次
          #readinessProbe: 
          #  httpGet:       
          #    port: 8761           # 必填
          #    path: /eureka/       # 选填，检查的路径，该容器是没有该路径的
          #    scheme: HTTP         # 默认是HTTP
          #  initialDelaySeconds: 10   
          #  timeoutSeconds: 2        
          #  periodSeconds:  3    # 在Pod的整个生命周期中运行，对故障敏感的服务，该值可以设小一点
          #  successThreshold: 1
          #  failureThreshold: 2
        livenessProbe: 
          tcpSocket: 
            port: 8761
          initialDelaySeconds: 10   
          timeoutSeconds: 2        
          periodSeconds:  3    # 在Pod的整个生命周期中运行，对故障敏感的服务，该值可以设小一点     
          successThreshold: 1
          failureThreshold: 2
          
[root@master 1.3]# kubectl apply -f 4.eureka_sts.yaml
[root@master 1.3]# kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
eureka-0   1/1     Running   0          6m54s
eureka-1   1/1     Running   0          6m28s
eureka-2   1/1     Running   0          6m3s

# 这个时候我们要让外部能访问我们的集群，这时可以再创建一个svc来暴露服务
[root@master 1.3]# cat 5.eureka_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: eureka-svc
spec:
  selector:
    app: eureka  # 这里要匹配sts定义的Pod的标签，写错就会导致代理不到后端的Pod
  type: NodePort # 每个节点都会暴露一个高位端口，供外部访问
  ports: 
    - name: eureka
      protocol: TCP
      port: 8761
      targetPort: 8761
status:
  loadBalancer: {}
[root@master 1.3]# kubectl apply -f 5.eureka_svc.yaml
```

![image-20260104180419473](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260104180419473.png)

这个时候就可以通过每个节点的IP+31877端口访问我们的 Eureka 集群了

![image-20260104180726399](https://oldhe.oss-cn-hangzhou.aliyuncs.com/typora_images/image-20260104180726399.png)
