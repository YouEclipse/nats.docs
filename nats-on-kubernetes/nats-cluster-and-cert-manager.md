# NATS 集群和证书管理

First we need to install the cert-manager component from [jetstack](https://github.com/jetstack/cert-manager):

首先我们需要从 [jetstack](https://github.com/jetstack/cert-manager) 安装 cert-manager 组件:

```text
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.0/cert-manager.yaml
```

If you are running Kubernetes &lt; 1.15, use `cert-manager-legacy.yaml` instead.

如果你使用 &lt; 1.15 版本的 Kubernetes，请使用 `cert-manager-legacy.yaml`

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: selfsigning
spec:
  selfSigned: {}
```

```text
clusterissuer.certmanager.k8s.io/selfsigning unchanged
```

Next, let's create the CA for the certs:

接下来,创建 CA:

```yaml
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: nats-ca
spec:
  secretName: nats-ca
  duration: 8736h # 1 year
  renewBefore: 240h # 10 days
  issuerRef:
    name: selfsigning
    kind: ClusterIssuer
  commonName: nats-ca
  usages:
  - cert sign
  organization:
  - Your organization
  isCA: true
---
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: nats-ca
spec:
  ca:
    secretName: nats-ca
```

现在创建客户端用于连接的DNS名称匹配的证书,在这个场景下，流量将在 Kuberntes 内部，所以我们使用实际上是 headless service `nats` 的DNS名称\(这个一个headless service 的[示例](https://github.com/nats-io/k8s/blob/master/nats-server/nats-server-plain.yml#L24-L47)\)

```yaml
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: nats-server-tls
spec:
  secretName: nats-server-tls
  duration: 2160h # 90 days
  renewBefore: 240h # 10 days
  issuerRef:
    name: nats-ca
    kind: Issuer
  usages:
  - signing
  - key encipherment
  - server auth
  organization:
  - Your organization
  commonName: nats.default.svc.cluster.local
  dnsNames:
  - nats.default.svc
```

In case of using the NATS operator, the Routes use a service named `$YOUR_CLUSTER-mgmt` \(this may change in the future\)

如果使用 NATS operator, 路由使用一个名叫 `$YOUR_CLUSTER-mgm` （未来可能会修改）的service

```yaml
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: nats-routes-tls
spec:
  secretName: nats-routes-tls
  duration: 2160h # 90 days
  renewBefore: 240h # 10 days
  issuerRef:
    name: nats-ca
    kind: Issuer
  usages:
  - signing
  - key encipherment
  - server auth
  - client auth
  organization:
  - Your organization
  commonName: "*.nats-mgmt.default.svc.cluster.local"
  dnsNames:
  - "*.nats-mgmt.default.svc"
```

Now let's create an example NATS cluster with the operator:

现在用 operator 创建一个NATS 集群的示例:

```yaml
apiVersion: "nats.io/v1alpha2"
kind: "NatsCluster"
metadata:
  name: "nats"
spec:
  # Number of nodes in the cluster
  size: 3
  version: "2.1.4"

  tls:
    # Certificates to secure the NATS client connections:
    serverSecret: "nats-server-tls"

    # Name of the CA in serverSecret
    serverSecretCAFileName: "ca.crt"

    # Name of the key in serverSecret
    serverSecretKeyFileName: "tls.key"

    # Name of the certificate in serverSecret
    serverSecretCertFileName: "tls.crt"

    # Certificates to secure the routes.
    routesSecret: "nats-routes-tls"

    # Name of the CA in routesSecret
    routesSecretCAFileName: "ca.crt"

    # Name of the key in routesSecret
    routesSecretKeyFileName: "tls.key"

    # Name of the certificate in routesSecret
    routesSecretCertFileName: "tls.crt"
```

确认所有的 pods 都已部署完成:

```bash
kubectl get pods -o wide
```

```bash
NAME     READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE
nats-1   1/1     Running   0          4s    172.17.0.8    minikube   <none>
nats-2   1/1     Running   0          3s    172.17.0.9    minikube   <none>
nats-3   1/1     Running   0          2s    172.17.0.10   minikube   <none>
```

查看日志:

```bash
kubectl logs nats-1
```

```text
[1] 2019/12/18 12:27:23.920417 [INF] Starting nats-server version 2.1.4
[1] 2019/12/18 12:27:23.920590 [INF] Git commit [not set]
[1] 2019/12/18 12:27:23.921024 [INF] Listening for client connections on 0.0.0.0:4222
[1] 2019/12/18 12:27:23.921047 [INF] Server id is NDA6JC3TGEADLLBEPFAQ4BN4PM3WBN237KIXVTFCY3JSTDOSRRVOJCXN
[1] 2019/12/18 12:27:23.921055 [INF] Server is ready
```

