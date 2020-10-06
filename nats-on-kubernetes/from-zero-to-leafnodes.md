# 使用Helm从0到 K8s到 子节点

首先，我们需要一些已经配置好的Kubernetes集群。在这个场景下我们将使用 `doctl`创建几个集群，当然你可以用任何可行的Kubernetes 集群搭建的解决方案。

```text
brew install doctl
doctl kubernetes cluster create nats-k8s-sfo2 --count 3 --region sfo2
doctl kubernetes cluster create nats-k8s-ams3 --count 3 --region ams3
```

接下来，获取启用了子节点的 NGS 凭据。 请遵循[以下说明](https://synadia.com/ngs/signup) 并选择免费的`Developer`计划，该计划将允许您创建多个集群间的子节点的连接。一旦你获得了这些凭据，将这些凭据作为密钥上传到k9s集群中。

```bash
for ctx in do-ams3-nats-k8s-ams3 do-sfo2-nats-k8s-sfo2; do
  kubectl --context $ctx create secret generic ngs-creds --from-file $HOME/.nkeys/creds/synadia/NGS/NGS.creds
done
```

安装 Helm3 然后添加 NATS 的 Helm chart 仓库:

```text
brew install helm
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo update
```

创建添加子节点 连接到 NGS 的配置:

```text
# nats.yaml
leafnodes:
  enabled: true
  remotes:
    - url: tls://connect.ngs.global:7422
      credentials:
        secret:
          name: ngs-creds
          key: NGS.creds
natsbox:
  enabled: true
```

将它部署到不同区域的k8s集群:

```bash
for ctx in do-ams3-nats-k8s-ams3 do-sfo2-nats-k8s-sfo2; do
  helm --kube-context $ctx install nats nats/nats -f nats.yaml
done
```

使用部署在各个集群的 nats-box 容器来测试跨区域的链接:

```text
kubectl --context do-ams3-nats-k8s-ams3  exec -it nats-box -- nats-sub -s nats hello
Listening on [hello]

while true; do
  kubectl --context do-sfo2-nats-k8s-sfo2  exec -it nats-box -- nats-pub -s nats hello 'Hello World!'
done
```

订阅的结果:

```text
[#1] Received on [hello]: 'Hello World!'
[#2] Received on [hello]: 'Hello World!'
[#3] Received on [hello]: 'Hello World!'
[#4] Received on [hello]: 'Hello World!'
[#5] Received on [hello]: 'Hello World!'
[#6] Received on [hello]: 'Hello World!'
[#7] Received on [hello]: 'Hello World!'
[#8] Received on [hello]: 'Hello World!'
[#9] Received on [hello]: 'Hello World!'
```

