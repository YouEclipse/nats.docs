# 创建一个 Kubernetes 集群

你将通过本文的示例，在不同的云服务产商创建包含3个节点的k8s集群来尝试 NATS。

## Google Kubernetes Engine

Use [gcloud](https://cloud.google.com/sdk/gcloud/) to create a 3 node [regional](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster) Kubernetes cluster on `us-west2`.

通过 [gcloud](https://cloud.google.com/sdk/gcloud/) 在 `us-west2` 区域创建包含3个节点的 [跨区域](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster) Kubernetes 集群

```bash
# Create a 3 node Kubernetes cluster. One node in each of the region's three zones.
gcloud container clusters create nats-k8s-cluster \
  --project $YOUR_GOOGLE_CLOUD_PROJECT \
  --region us-west2 \
  --num-nodes 1 \
  --machine-type n1-standard-2
```

Note that since this is a regional cluster we are specifying `--num-nodes 1` which will create a kubelet on 3 different zones. If you are creating a [single-zone cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster) but want 3 nodes then you have to specify `--num-nodes 3`.  

需要注意的一点是我们在创建集群的时候指定了 `--num-nodes 1`,这意味着我们将在三个不同的区域创建 kubelet。 如果你要创建的是 [单区域集群](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster) 并且包含3个节点，你需要指定 `--num-nodes 3`。

## Amazon Kubernetes Service

The [eksctl](https://github.com/weaveworks/eksctl) is a very helpful tool to manage EKS clusters, you can find more docs on how to set it up [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html). 

[eksctl](https://github.com/weaveworks/eksctl) 可以非常方便地创建 EKS 集群，你可以在[这里](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)找到更多配置的文档。

```bash
# Create 3 node Kubernetes cluster
eksctl create cluster --name nats-k8s-cluster \
  --nodes 3 \
  --node-type=t3.large \
  --region=eu-west-1

# Get the credentials for your cluster
eksctl utils write-kubeconfig --name $YOUR_EKS_NAME --region eu-west-1
```

## Digital Ocean

你可以使用 [doctl](https://github.com/digitalocean/doctl)  来创建集群:

```bash
doctl kubernetes cluster create nats-k8s-nyc2 --count 3 --region nyc1
```

## Azure Kubernetes Service

Using [az](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) you can create a cluster like this:

使用 [az](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)  你可以像这样创建一个集群:

```bash
# In case not done already, register to use some services:
az login
az provider register -n Microsoft.Network
az provider register -n Microsoft.Storage
az provider register -n Microsoft.Compute
az provider register -n Microsoft.ContainerService

# Create resource group and 3 node cluster
az group create --name nats --location westus
az aks create   --resource-group nats --name nats  --node-count 3 --node-vm-size Standard_DS1_v2
az aks get-credentials --resource-group nats --name nats
```

_Note_ In order to be able to access NATS externally you need to provision public IPs for your cluster installing the following component [dgkanatsios/AksNodePublicIPController](https://github.com/dgkanatsios/AksNodePublicIPController):

_需要注意的是_ 为了能够从外部访问 NATS,你需要提供公共 IP 来为你的集群安装以下组件[dgkanatsios/AksNodePublicIPController](https://github.com/dgkanatsios/AksNodePublicIPController):

```bash
kubectl create -n kube-system -f https://raw.githubusercontent.com/dgkanatsios/AksNodePublicIPController/7846c78f77dc5cd4b43629bb5cb7ff3818594aee/deploy.yaml
```

在这些组件安装完成后，你的\(Kubernetes\)集群将 为 NATS 提供用于向客户端广播的内部IP:

```text
kubectl get nodes -o wide
NAME                       STATUS   ROLES   AGE     VERSION    INTERNAL-IP   EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-nodepool1-18657977-0   Ready    agent   5d16h   v1.13.12   10.240.0.6    52.191.186.114   Ubuntu 16.04.6 LTS   4.15.0-1060-azure   docker://3.0.7
aks-nodepool1-18657977-1   Ready    agent   5d17h   v1.13.12   10.240.0.4    52.229.11.82     Ubuntu 16.04.6 LTS   4.15.0-1060-azure   docker://3.0.7
aks-nodepool1-18657977-2   Ready    agent   5d17h   v1.13.12   10.240.0.5    13.77.149.235    Ubuntu 16.04.6 LTS   4.15.0-1060-azure   docker://3.0.7
```

