# NATS 和 Prometheus Operator

## 安装 Operator

安装 NATS Operator:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/nats-io/nats-operator/master/deploy/00-prereqs.yaml
$ kubectl apply -f https://raw.githubusercontent.com/nats-io/nats-operator/master/deploy/10-deployment.yaml
```

安装包含 RBAC 资源定义\(prometheus-operator service account\)的的 Prometheus Operator：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: prometheus-operator
    app.kubernetes.io/version: v0.30.0
  name: prometheus-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-operator
subjects:
- kind: ServiceAccount
  name: prometheus-operator
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: prometheus-operator
    app.kubernetes.io/version: v0.30.0
  name: prometheus-operator
rules:
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs:
  - '*'
- apiGroups:
  - monitoring.coreos.com
  resources:
  - alertmanagers
  - prometheuses
  - prometheuses/finalizers
  - alertmanagers/finalizers
  - servicemonitors
  - podmonitors
  - prometheusrules
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
  - delete
- apiGroups:
  - ""
  resources:
  - services
  - services/finalizers
  - endpoints
  verbs:
  - get
  - create
  - update
  - delete
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: prometheus-operator
    app.kubernetes.io/version: v0.30.0
  name: prometheus-operator
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/name: prometheus-operator
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/name: prometheus-operator
        app.kubernetes.io/version: v0.30.0
    spec:
      containers:
      - args:
        - --kubelet-service=kube-system/kubelet
        - --logtostderr=true
        - --config-reloader-image=quay.io/coreos/configmap-reload:v0.0.1
        - --prometheus-config-reloader=quay.io/coreos/prometheus-config-reloader:v0.30.0
        image: quay.io/coreos/prometheus-operator:v0.30.0
        name: prometheus-operator
        ports:
        - containerPort: 8080
          name: http
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-operator
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: prometheus-operator
    app.kubernetes.io/version: v0.30.0
  name: prometheus-operator
  namespace: default
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: prometheus-operator
    app.kubernetes.io/version: v0.30.0
  name: prometheus-operator
  namespace: default
spec:
  clusterIP: None
  ports:
  - name: http
    port: 8080
    targetPort: http
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: prometheus-operator
```

##  创建一个NATS 集群实例

```yaml
apiVersion: "nats.io/v1alpha2"
kind: "NatsCluster"
metadata:
  name: "nats-cluster"
spec:
  size: 3
  version: "1.4.1"
  pod:
    enableMetrics: true
    metricsImage: "synadia/prometheus-nats-exporter"
    metricsImageTag: "0.3.0"
```

## 创建一个 Prometheus 实例

### 为 Prometheus 实例创建RBAC资源 \(ServiceAccount\)

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```

### 创建 Prometheus 实例

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      app: nats
      nats_cluster: nats-cluster
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: true
```

## 为 NATS cluster 创建 ServiceMonitor 资源

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nats-cluster
  labels:
    app: nats
    nats_cluster: nats-cluster
spec:
  selector:
    matchLabels:
      app: nats
      nats_cluster: nats-cluster
  endpoints:
  - port: metrics
```

## 验证

```text
kubectl port-forward prometheus-prometheus-0 9090:9090
```

### 验证结果

![](https://user-images.githubusercontent.com/26195/59470419-2066fd80-8e27-11e9-9e3e-250296a091da.png)

