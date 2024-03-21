# kube-proxy 详解

kube-proxy 是 worker 节点的核心组件之一(另一个 kubelet)，主要作用是负责整个集群的服务发现以及负载均衡；
kube-proxy 早期还有会承担流量转发的功能，但由于转发比较低效且耗性能，现已废弃。

## 工作模型

Kubernetes中的Pods是临时的，可随时被终止或重启。由于这种行为，我们不能依赖于它们的IP地址，因为它们总是在变。

于是引入了 Service 对象，它为 Pods 提供一个稳定的虚拟 IP 地址，用于连接Pods。每个Service与一组Pods相关联，当流量到达Service时，根据 NAT 规则将其重定向到相应的后端Pods。
kube-proxy 就是用于管理 Service -> Pod 的转发规则，同时提供负载均衡。

kube-proxy安装在每个节点中，它监视与Service对象及其端点相关的更改，然后将这些更改转换为节点内的实际网络规则。
kube-proxy通常以 DaemonSet 的形式在集群中运行，但也可直接作为节点上的Linux进程安装。

在 Linux 系统中，NAT 转发的规则是通过 iptables 来实现与维护的。

**小结**

kube-proxy 部署在每个节点上面，通过 informer 机制监听所有 service 对象的变化，包括与之与对应的 pods 的变化。
将监听的变化维护成一个 ClusterIP -> PodIP 的对应关系表，同时通过 iptables 体现到每个节点的转发规则上。

## k8s 网络模型

k8s 集群的网络模型可抽象为4层，分别为：主机网络、容器网络、Service网络、外部网络(Ingress)。

- 主机网络：节点间互通互联的网络，公有云上体现为公网 IP
- 容器网络：pod 与 pod、主机与 pod 间互通互联的网络，通常通过 flannel、calico 等实现 CNI 网络的组件来实现，通常来说是在主机网络上实现一个overlay网络
- Service网络：Service 到 Pods 的互通互联，也即 ClusterIP -> PodIP
- 外部网络：集群外部访问集群内部服务，通常通过 Ingress-Controller + DaemonSet 实现节点本地端口监听，配合外部 DNS/SLB实现集群外请求的接入

## 原理 & 实践

kube-proxy是 ClusterIP 到 PodIP 的映射。kube-dns实现服务名到 ClusterIP 的映射。

如下示例展示了 iptables 是如何进行转发的。


1. 创建  deployment service
```shell
# 应用配置 创建对象
kubectl apply -f nginx-demo.yaml
```
```shell
# 查看 pods
kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-849db79549-6h588   1/1     Running   0          22h
nginx-demo-849db79549-5xkm6   1/1     Running   0          22h
```
```shell
# 查看 service 
kubectl get svc
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes       ClusterIP   10.43.0.1      <none>        443/TCP   5d9h
nginx-demo-svc   ClusterIP   10.43.171.67   <none>        80/TCP    90m
```
```shell
# 查看endpoints
kubectl get ep
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes       ClusterIP   10.43.0.1      <none>        443/TCP   5d9h
nginx-demo-svc   ClusterIP   10.43.171.67   <none>        80/TCP    90m
```

2. 查看 iptables 路由规则
```shell
# 首先查看 PREROUTING 的 nat 规则
iptables -t nat -L PREROUTING
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

# 查看 KUBE-SERVICES
iptables -t nat -L KUBE-SERVICES |grep 10.43.171.67
KUBE-SVC-4UYISBIO2TCCKNCL  tcp  --  anywhere             10.43.171.67         /* default/nginx-demo-svc cluster IP */ tcp dpt:http

# 查看 KUBE-SVC-4UYISBIO2TCCKNCL
iptables -t nat -L KUBE-SVC-4UYISBIO2TCCKNCL
Chain KUBE-SVC-4UYISBIO2TCCKNCL (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  tcp  -- !aliyun-master-120-79-152-224/16  10.43.171.67         /* default/nginx-demo-svc cluster IP */ tcp dpt:http
KUBE-SEP-N73DLKIA7YOHXVF4  all  --  anywhere             anywhere             /* default/nginx-demo-svc -> 10.42.1.20:80 */ statistic mode random probability 0.50000000000
KUBE-SEP-Q4OWDBB5ABQUHTSD  all  --  anywhere             anywhere             /* default/nginx-demo-svc -> 10.42.2.18:80 */

#
iptables -t nat -L KUBE-SEP-N73DLKIA7YOHXVF4
Chain KUBE-SEP-N73DLKIA7YOHXVF4 (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  10.42.1.20           anywhere             /* default/nginx-demo-svc */
DNAT       tcp  --  anywhere             anywhere             /* default/nginx-demo-svc */ tcp to:10.42.1.20:80 
```

### 参考

- [深入理解K8S网络原理](https://blog.csdn.net/meser88/article/details/131192724)
- [Kubernetes核心组件之kube-proxy实现原理](https://cloud.tencent.com/developer/article/2377930)
- [iptables零基础快速入门系列](https://www.zsythink.net/archives/tag/iptables/)
- [Linux利用iptables实现负载均衡](https://blog.csdn.net/ksj367043706/article/details/89764546)
- 