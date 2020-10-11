---
title: "Docker和Kubernetes网络模型"
category: Distributed System
tag: [docker, kubernetes]
---
## Docker网络模型
### Bridge模式（默认）
Docker程序启动后会创建一个bridge0网桥，并分配一个IP，可以想象成一个虚拟的交换机，创建的容器实例都会通过虚拟网卡veth pair设备连接到这个网桥上，并以网桥IP作为网关，这样就可以实现容器间的通信。
如果容器内程序想访问宿主机服务，则可以直接访问bridge0的IP，注意：此流量不走localhost，宿主机上服务需要监听0.0.0.0,并且此方法在不同系统间不通用。
如果容器内程序想访问外网服务，则需要通过SNAT机制，将来自容器内部的源IP替换成宿主机IP。
如果外部程序想访问容器内服务，则需要在启动容器时做端口绑定,也是通过iptables添加一条DNAT规则，实现目的IP转换。如：
```
docker run -tid --name db -p 3306:3306 MySQL
在宿主机上，可以通过iptables -t nat -L -n，查到一条DNAT规则：
 DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:3306 to:172.17.0.5:3306
```
### Host模式
该模式下容器网络并未与宿主机网络进行隔离，而是占用了宿主机的端口，容易造成端口冲突。

### Containner模式
该模式指后创建的容器与已有容器共用同一个Network Namespace，即共用一个IP和端口范围。则两个容器可以通过localhost网卡通信。

## Kubernetes网络模型
### Pod内容器间的通信
每个Pod创建的时候都先会创建一个默认的Pause容器，后面每个新建的容器则都使用Containner模式，这样所有容器之间都可以通过localhost网卡通信，实现了Pod内容器的网络共享。

### 同一Node上Pod间的通信
采用了类似Docker Bridge模式的网桥技术，将不同Pod连接在了同一网段中。

### 不同Node上Pod间的通信
Kubernetes中规定Pod IP在整个集群中唯一，且互相可以不通过IP转换直接通信。此点相比于Docker简化了跨Node通信的配置工作。
此时需要借助实现了CNI标准的插件来实现了，比如：flannel、weave等。
flannel为Kubernetes集群提供了一个三层路由转发网络，通过存储在etcd的pod ip与node ip对应关系，转发数据包到相应的节点上。

### Service与Pod间的通信
由于Pod IP会经常变化，所以K8s在提供同样服务的Pod之上又抽象出了Service的概念。Service具有固定的CluserIp：port，可以通过内建的负载均衡器路由到背后的pod容器。
在Pod内程序往Service上发送数据包时，数据包会经过iptables进行过滤，iptables接受数据包后会使用kube-proxy在Node上安装的规则来响应Service或Pod的事件，
将数据包的目的地址从Service的IP重写为Service后端特定的Pod IP，之后就与不同Node上Pod间的通信过程类似。

### 外部网络访问Service
#### NodePort模式+LoadBalancer(四层流量入口)
通过配置Service类型为NodePort，则外网则可以通过集群中任意节点的NodeIp:NodePort访问服务，接收到报文的节点会重定向到对应的Service。
NodePort是靠kube-proxy服务通过iptables的nat转换功能实现的，kube-proxy会在运行过程中动态创建与Service相关的iptables规则，这些规则实现了NodePort的请求流量重定向到kube-proxy进程上对应的Service的代理端口上。
kube-proxy接受到Service的请求访问后，会从service对应的后端Pod中选择一个进行访问。
在这之前如果可以添加一个负载均衡器，比如目前阿里云上的SLB，则请求会通过访问负载均衡器的IP在被负载到K8s集群中的Node上。

### Ingress模式（七层流量入口）
不像负载均衡器每个服务需要一个公开ip,ingress所有服务只需要一个公网ip,当客户端向Ingress发送http请求时候，ingress会根据请求的主机名和路径决定请求转发到那个服务。
它底层其实是通过Nginx来实现的，它会去对应服务的Endpoints列表里查找pod IP自己选择一个直接访问。
基于http请求的负责模式可以通过host和url进行请求细分服务流量，还可以在请求的X-Forwarded-For标头中提供原始客户端的IP地址。

[Docker容器是如何进行网络通信的](https://zhuanlan.zhihu.com/p/104503057)
[K8s网络模型](http://www.spring4all.com/article/6886)
