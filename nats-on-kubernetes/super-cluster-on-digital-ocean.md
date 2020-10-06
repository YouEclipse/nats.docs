# 使用Helm在Digital Ocean 创建一个NATS 超级集群

我们使用NATS Gateway创建一个超级集群。首先我们在 NYC, Amsterdam 和 San Francisco这3个区域创建3个不同的集群:

```bash
doctl kubernetes cluster create nats-k8s-nyc1 --count 3 --region nyc1
doctl kubernetes cluster create nats-k8s-sfo2 --count 3 --region sfo2
doctl kubernetes cluster create nats-k8s-ams3 --count 3 --region ams3
```

接下来，将三个区域的防火墙打开以便能够访问客户端，子节点和网关的的端口:

```bash
for firewall in `doctl compute firewall list | tail -n 3 | awk '{print $1}'`; do
  doctl compute firewall add-rules $firewall --inbound-rules protocol:tcp,ports:4222,address:0.0.0.0/0
  doctl compute firewall add-rules $firewall --inbound-rules protocol:tcp,ports:7422,address:0.0.0.0/0
  doctl compute firewall add-rules $firewall --inbound-rules protocol:tcp,ports:7522,address:0.0.0.0/0
done
```

在这个配置中，我们将会使用三个进群节点的外部IP来创建一个超级集群。对于生产环境的配置，我们推荐为每个服务创建一个 DNS 条目和 A记录。

```bash
for ctx in do-ams3-nats-k8s-ams3 do-nyc1-nats-k8s-nyc1 do-sfo2-nats-k8s-sfo2; do
  echo "name: $ctx"
  for externalIP in `kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'`; do 
    echo "- nats://$externalIP:7522"; 
  done
  echo
done
```

三个集群的Helm的定义如下：

```yaml
# super-cluster.yaml
nats:
  externalAccess: true
  logging:
    debug: false
    trace: false

cluster:
  enabled: true

gateway:
  enabled: true

  # NOTE: defined via --set gateway.name="$ctx"
  # name: $ctx

  gateways:
  - name: do-ams3-nats-k8s-ams3
    urls:
    - nats://142.93.251.181:7522
    - nats://161.35.12.245:7522
    - nats://161.35.2.153:7522

  - name: do-nyc1-nats-k8s-nyc1
    urls:
    - nats://142.93.251.181:7522
    - nats://161.35.12.245:7522
    - nats://161.35.2.153:7522

  - name: do-sfo2-nats-k8s-sfo2
    urls:
    - nats://142.93.251.181:7522
    - nats://161.35.12.245:7522
    - nats://161.35.2.153:7522

natsbox:
  enabled: true
```

使用集群名字作为网关名字通过Helm来部署超级集群:

```bash
for ctx in do-ams3-nats-k8s-ams3 do-nyc1-nats-k8s-nyc1 do-sfo2-nats-k8s-sfo2; do
  helm --kube-context $ctx install nats nats/nats -f super-cluster.yaml --set gateway.name=$ctx
done
```

配置完毕！现在可以跨区域发布一些消息了。

```bash
# Start subscription in Amsterdam
nats-box:~# kubectl --context do-ams3-nats-k8s-ams3 exec -it nats-box -- /bin/sh -l
nats-box:~# nats-sub -s nats hello

# Send messages from San Francisco region
nats-box:~# kubectl --context do-sfo2-nats-k8s-sfo2 exec -it nats-box -- /bin/sh -l
nats-box:~# nats-pub -s nats hello 'Hello World!'

# From outside of k8s can use the external IPs
$ nats-sub -s 142.93.251.181 hello
$ nats-pub -s 161.35.2.153 hello 'Hello World!'
```

