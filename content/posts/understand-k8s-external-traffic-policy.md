+++
title = "一个小实验理解 Kubernetes 的外部流量策略 (externalTrafficPolicy)"
lastmod = 2025-09-22T22:33:20+08:00
tags = ["kubernetes"]
draft = false
+++

> 关于 Service 的 `externalTrafficPolicy`, 官方文档的解释感觉有点模糊，看了网上其他人的解释也总感觉没有说到点子上。本文通过一个小实验，帮助你真正理解这个字段的含义和效果，以及相关注意事项。最后，本文总结了两种策略的优缺点。

在 Kubernetes 中，针对 Service 的外部流量有两种分发策略：

Cluster
: 节点在收到外部流量后，可以转发给集群中的任意节点（包括自己和其他节点）。

Local
: 节点在收到外部流量后，只会发送到当前节点的 Pod 中。如果当前节点中没有该 Service 对应的处于 ready 状态的后端 endpoint，则 **该流量会被丢弃** 。

上面关于两种分发策略的描述，经过我的实验验证并总结得出。

下面就介绍一下这个小实验，来帮助我们理解和证实上述结论。


## 实验环境 {#实验环境}

环境很简单，布署一个 k8s 集群，包含两个 worker 节点：

| NAME | STATUS | AGE  | VERSION      | INTERNAL-IP  |
|------|--------|------|--------------|--------------|
| gz1  | Ready  | 4d2h | v1.33.4+k3s1 | 192.168.8.1  |
| xjp1 | Ready  | 3d8h | v1.33.4+k3s1 | 172.17.26.70 |


## 实验步骤 {#实验步骤}


### 创建 traffic-test deployment {#创建-traffic-test-deployment}

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-test
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: traffic-test
        template:
          metadata:
            labels:
              app: traffic-test
              spec:
                containers:
                  - name: whoami
                    image: traefik/whoami
                    ports:
                      - containerPort: 80
```

注意，这里的 replicas=1, 因此只会有一个 pod 部署在其中一个 node 上：

```shell
kubectl get pods -lapp=traffic-test -o wide
```

```text
NAME                            READY   STATUS    RESTARTS   AGE    IP           NODE   NOMINATED NODE   READINESS GATES
traffic-test-7787c8cc97-pm6k8   1/1     Running   0          107m   10.42.1.59   xjp1   <none>           <none>
```

可以看到，该 pod 部署在 node xjp1 上。


### 创建 traffic-test service {#创建-traffic-test-service}

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traffic-test
  spec:
    selector:
      app: traffic-test
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
          type: NodePort
          externalTrafficPolicy: Cluster
```

注意：

-   Service 类型设置为 NodePort 类型

    > `externalTrafficPolicy` 仅针对外部流量有意义，而只有 NodePort, LoadBalancer
    > 这两种类型的 Service 才会直接接收外部流量。简单起见，这里使用 NodePort 类型来测试。

-   `externalTrafficPolicy` 设置为 Cluster

查看一下 Service 的 NodePort 端口号：

```shell
kubectl get svc traffic-test
```

```text
NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
traffic-test   NodePort   10.43.203.41   <none>        80:30316/TCP   104m
```

可以看到， service 映射到了主机的 30316 端口上。后面我们可以通过地址
`nodeIP:30316` 来访问该服务。


### 测试 Cluster 流量策略 {#测试-cluster-流量策略}

由于该服务为 NodePort 类型，且 `externalTrafficPolicy` 设置为 Cluster, 那么按照 Cluster 的定义，我们应该可以在集群外部，通过集群中任意节点的 IP 访问到该服务。

我们测试一下。

首先尝试通过 xjp1 节点访问该 service (xjp1 即 pod 所在的节点，参考[实验环境](#实验环境),
其 IP 为 172.17.26.70)：

```shell
curl -s 172.17.26.70:30316
```

```text
Hostname: traffic-test-7787c8cc97-pm6k8
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.59
IP: fe80::d496:16ff:fe3d:5e74
RemoteAddr: 10.42.1.1:25389
GET / HTTP/1.1
Host: 172.17.26.70:30316
User-Agent: curl/8.7.1
Accept: */*

```

✅ xjp1 节点测试成功。

接下来尝试通过 gz1 访问，参考[实验环境](#实验环境), 其 IP 为 192.168.8.1：

```shell
curl -s 192.168.8.1:30316
```

```text
Hostname: traffic-test-7787c8cc97-pm6k8
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.59
IP: fe80::d496:16ff:fe3d:5e74
RemoteAddr: 10.42.0.0:18798
GET / HTTP/1.1
Host: 192.168.8.1:30316
User-Agent: curl/8.7.1
Accept: */*

```

✅ gz1 节点测试成功。

有两点值得注意：

-   gz1 节点上并未部署该 service 对应的后端 pod, 但仍然可以通过该节点将流量路由到其后端 endpoint, 说明 Cluster 策略生效了。

-   两次测试返回的 RemoteAddr 都是集群内部的 IP, 而不是客户端 IP。这是因为流量可能需要经过 node 转发，所以会统一做一次 SNAT, 将流量的源 IP 地址修改为
    node 的 IP 地址。因此， **会丢失原始客户端的源 IP 地址** 。


### 测试 Local 流量策略 {#测试-local-流量策略}

首先将服务的 `.spec.externalTrafficPolicy` 修改成 Local 类型：

```shell
kubectl patch svc traffic-test -p '{"spec": {"externalTrafficPolicy": "Local"}}'
```

```text
service/traffic-test patched
```

```shell
kubectl get svc traffic-test -o yaml | yq '.spec.externalTrafficPolicy'
```

```text
Local
```

确认已经修改成 Local 了。

重复上面两个测试。

测试 xjp1:

```shell
curl -s 172.17.26.70:30316
```

```text
Hostname: traffic-test-7787c8cc97-pm6k8
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.59
IP: fe80::d496:16ff:fe3d:5e74
RemoteAddr: 192.168.22.2:61899
GET / HTTP/1.1
Host: 172.17.26.70:30316
User-Agent: curl/8.7.1
Accept: */*

```

✅ xjp1 节点测试成功。而且，RemoteAddr 获取正确，正是真实的客户端 IP 地址。

测试 gz1:

```shell
curl -s --connect-timeout 1 192.168.8.1:30316 || echo timeout
```

```text
timeout
```

❌ gz1 节点测试失败。

这是符合预期的，因为 gz1 上面没有 service traffic-test 对应的后端 endpoint, 在外部流量策略为 Local 的情况下，流量会被丢弃。这印证了我们文章开头的结论。


## 总结 {#总结}

Cluster
: 节点在收到外部流量后，可以转发给集群中的任意节点（包括自己和其他节点）。

    优点：

    -   负载均衡： Cluster 的分发策略可以确保流量在不同的后端负载之间均衡分发。

    -   高可用性： 即使某个节点出现故障，只要还有其他节点有部署该 service 对应的
        endpoint, 便可以提供服务。

    缺点：

    -   性能开销：需要做一次 SNAT，并且可能导致第二跳 (second hop) 到另一个节点，性能上略有影响。

    -   丢失客户端 IP：由于做了一次 SNAT, 后端 pod 无法获得客户端的真实 IP 地址。


Local
: 节点在收到外部流量后，只会发送到当前节点的 Pod 中。如果当前节点中没有该 Service 对应的处于 ready 状态的后端 endpoint，则 **该流量会被丢弃** 。

    优点：

    -   性能：相比 Cluster 策略，由于不需要做 SNAT 且不存在第二跳，性能上更有优势。

    -   保留客户端 IP：由于不需要做 SNAT，客户端 IP 得以保留。

    缺点：

    -   负载均衡： 依赖外部负载均衡器的实现，由于外部负载均衡器不知道每个节点上作为目标的 pod 数量，因此较难保证负载均衡。

    -   高可用性： 如果某个节点收到流量，但该节点没有部署对应的目标 pod，或者该 pod
        出现故障，则会导致服务不可用。

    -   丢失客户端IP：由于做了一次 SNAT, 后端 pod 无法获得客户端的真实 IP 地址。
