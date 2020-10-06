# 安装 NATS 和 NATS Streaming

## NATS 和 NATS Streaming 的最小化安装

你可以通过以下方式来安装包含最基本组件的NATS:

```bash
# Single server NATS
kubectl apply -f https://raw.githubusercontent.com/nats-io/k8s/master/nats-server/single-server-nats.yml

kubectl apply -f https://raw.githubusercontent.com/nats-io/k8s/master/nats-streaming-server/single-server-stan.yml
```

这将会安装:

* 一个包含单个 NATS server\(不支持授权和TLS\) 的 statefulset 
* 一个你可以访问的 `nats`headless service
* 一个在 `stan` 集群下的 NATS Streaming 服务

需要注意的是，你的应用只需要访问 `nats`service，当然，NATS Streaming 服务也将使用 NATS 来通信.

接下来,可以尝试使用 `nats-box`  来连接`nats` service，以确保 NATS 和 NATS Streaming 已经配置好。

```bash
kubectl run -i --rm --tty nats-box --image=synadia/nats-box --restart=Never

# Send message to NATS
nats-box:~# nats-sub -s nats hello &
Listening on [hello]

nats-box:~# nats-pub -s nats hello world
# Send/Receive message to STAN
nats-box:~# stan-pub -s nats -c stan hello world
Published [hello] : 'world'

nats-box:~# stan-sub -s nats -c stan hello
Connected to nats clusterID: [stan] clientID: [stan-sub]
Listening on [hello], clientID=[stan-sub], qgroup=[] durable=[]
[#1] Received: sequence:1 subject:"hello" data:"world" timestamp:1579544643374163630
```

## 使用 StatefulSets 来配置高可用

为了高可用，你可以将 NATS 和NATS Streaming\(STAN\) 配置成集群模式使用。下面的命令将创建一个 包含3个NATS节点和3个NATS Streaming节点的集群，并且挂载了一个持久卷。需要注意的是，为了能正常工作，你的Kubernetes集群需要超过一个node;因此如果需要部署在ninikube或者docker客户端，请使用单节点的安装方式。

```bash
# Create NATS cluster
kubectl apply -f https://raw.githubusercontent.com/nats-io/k8s/master/nats-server/simple-nats.yml

# Create STAN cluster
kubectl apply -f https://raw.githubusercontent.com/nats-io/k8s/master/nats-streaming-server/simple-stan.yml
```

对于NATS Streaming，我们比较推荐使用容错模式\(Fault Tolerance mode\),因为相比集群模式，它有更好的性能以及更好的failover机制。你可以参考这篇文档 [容错\(Fault Tolerance\)模式下的NATS Streaming Cluster](stan-ft-k8s-aws.md)。

## 使用 Helm Charts

对于 NATS 和 NATS Streaming 都提供官方的 Helm Chart。

* [NATS Helm Chart](https://github.com/nats-io/k8s/tree/master/helm/charts/nats)
* [NATS Streaming Helm Chart](https://github.com/nats-io/k8s/tree/master/helm/charts/stan)

