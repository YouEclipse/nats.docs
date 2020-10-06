# 使用负载均衡器\(Load Balancer\) 为NATS提供外部访问

虽然并不推荐使用负载均衡器为NATS提供外部访问，但是有时候由于一些策略的原因，使用负载均衡器也未尝不可。在这种场景下，使用支持TCP的四层负载均衡器则是选择之一。

在下面的示例中，你可以找到如何使用一个 [AWS Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html) 从外部连接到一个配置好TLS的NATS 集群中。

```bash
# One-line installer creates a secure cluster named 'nats'
$ curl -sSL https://nats-io.github.io/k8s/setup.sh | sh

# Create AWS Network Load Balancer service
$ echo '
apiVersion: v1
kind: Service
metadata:
  name: nats-nlb
  namespace: default
  labels:
    app: nats
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
  - name: nats
    port: 4222
    protocol: TCP
    targetPort: 4222
  selector:
    app: nats
' | kubectl apply -f -

$ kubectl get svc nats-nlb -o wide
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP                                                                     PORT(S)          AGE    SELECTOR
nats-nlb   LoadBalancer   10.100.67.123   a18b60a948fc611eaa7840286c60df32-9e96a2af4b5675ec.elb.us-east-2.amazonaws.com   4222:30297/TCP   151m   app=nats

$ nats-pub -s nats://a18b60a948fc611eaa7840286c60df32-9e96a2af4b5675ec.elb.us-east-2.amazonaws.com:4222 -creds nsc/nkeys/creds/KO/A/test.creds test.foo bar
```

另外，我们推荐将 [no\_advertise](../nats-server/configuration/clustering/cluster_config.md)  设置成`true`来避免将Kubernetes 中 pod的内部地址经过流言传播协议暴露给NATS 客户端。

