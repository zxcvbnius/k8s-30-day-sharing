# 前言
在今天的學習筆記中，將介紹 Kubernetes 的調度中心 **Master Node**。在 **Master Node** 上有著四種不同的元件：[etcd](https://github.com/coreos/etcd)，[Controller Manager](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#kubernetes-controller-manager-server)，[Scheduler](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#scheduler) 與 [API Server](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#kubernetes-api-server)。如官方提供的架構圖所示(下圖)，Kubernetes 可以分成兩個面向，一個是 Master Node (左半部)，另一部份則是過去二十幾天經常接觸到的 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 元件。在今天的學習筆記，我們將從開發者輸入 `kubectl create` 指令出發，簡單介紹 Master Node 內部是如何運作的。

![](https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.1/docs/design/architecture.png)
(擷自 Kubernetes Github 專案)

# Start from kuberctl create
在過去幾天的實作中，我們常用 **kubectl create** 指令在 Kubernetes 上創建一個新物件。今天我們將透過創建一個 Deployment 物件的指令 

```
$ kubectl create -f ./hello-deployment.yaml
```

開始說起....

```
# hello-deployment.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld-pod
  template:
    metadata:
      labels:
        app: helloworld-pod
    spec:
      containers:
      - name: my-pod
        image: zxcvbnius/docker-demo:latest
        ports:
        - containerPort: 3000
```


# Etcd
當我們在終端機輸入上述指令後，`kubectl` 的請求會送往 Kubernetes Cluster 中的 Master Node。Master 中的 API Server 接收到該請求之前，會先經過一層認證(authorization)，確認傳送方的身份沒問題後，再將這個請求傳遞給 Master Node 中的 [API Server](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#kubernetes-api-server)， [API Server](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#kubernetes-api-server) 每次收到指令後，會先把每個接收到的請求內容存放在 [etcd](https://github.com/coreos/etcd) 中。

在 Kubernetes 中，[etcd](https://github.com/coreos/etcd) 用來存放 Cluster 中所有的 data。當 master nodes 因為某些原因而故障時，我們可以透過 [etcd](https://github.com/coreos/etcd) 幫我們還原 Cluster 的狀態。通常在實際場景中，我們偏好使用 [Multi-node etcd cluster](https://github.com/coreos/etcd) 勝過單一的 [etcd](https://github.com/coreos/etcd)，以防 [etcd](https://github.com/coreos/etcd) 節點本身也故障的狀況發生。以 Kubernetes 為例，官網建議 3 或 5 個的節點作為 [etcd](https://github.com/coreos/etcd) cluster。

# Controller Manager
除了我們先前介紹過的 [Node Controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day28)、 [Replication Controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day08)、[Endpoint Controller](https://kubernetes.io/docs/concepts/services-networking/service/) 外，Kubernetes 還有 [Namespace Controller](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)、[ServiceAccount Controller](https://kubernetes.io/docs/admin/service-accounts-admin/)等多種不同類型的 Controller。而 [Controller Manager](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#kubernetes-controller-manager-server) 則是 Kubernetes 中所有 Controllers 的核心管理者。**Controller Manager 會定期去訪問 API Server**，若有接收到變更的指令 Controller Manager 則會去更改這些 Controllers 的狀態。  

以上述 `kubectl create ...` 為例，Controller Manager 從 API Server 那邊得知需要創建一個新的 `hello-pod` 物件後會先經過一連串的檢查。在資源都許可的狀況下，Controller Manager 會建立一個新的 Pod 物件。

# [Scheduler](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#scheduler)
**除了 Controller Manager, Master Node 上的 Scheduler 也會定期去訪問 API Server**。若發現 Controller Manager 有新建置的 Pod 物件時，會負責將這些 Pod 安置在某一個 Node 上。若讀者對[第六天的學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day06)還有些印象，會知道每個 Node 裡面都會有一個 `kubelet`，**kubelet 相當於 node agent，做為 master node 與 node 即時溝通的橋樑**。因此，API Server 保有每個 Node 目前的最新狀況，而 [Scheduler](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md#scheduler) 可以根據 API Server 上每個 Node 目前的狀況，透過**特定的調度邏輯**將 Pod 放置在最適合的 Node 上。  

# Check pod status
最後，當我們想知道目前該 Pod 運行的狀況以及在哪個 Node 上運行時，我們只需要輸入 `kubectl get...` ，如此 Master Node 上擁有所有節點資訊的 API Server 收到 `get` 指令後，便會將目前該物件的狀態回傳給我們。

# 總結
今天的學習筆記中，雖然今天的學習筆記中沒有針對 Master Node 中的各個元件做進一步的分享，然而，希望透過我們熟悉的 `kubectl` 指令能快速的讓大家對 Master Node 的四個元件有基礎認識。Kubernetes 的 [Github](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/architecture.md) 與 [官網](https://kubernetes.io/docs/concepts/architecture/master-node-communication/) 對 Master Node 的架構以及如何與 Node 之間的溝通有著非常詳細的說明，網路上也可以找到許多關於 Kubernetes Master 的介紹，相信讀者花時間閱讀都會有不少的收穫唷。

# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
- [Kubernetes Github](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design)
