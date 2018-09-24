# 前言
在[前一天的學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day26)中提到，當 Kubernetes 提供給越來越多人使用時，我們可以透過 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 來避免單一 Pod 佔領整個 Cluster 的資源。今天的學習筆記中，想介紹 Kubernetes 的另一個元件 - [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)，今日筆記內容如下：

- 介紹什麼是 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- 實作：限制某一 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 的運算資源

# [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 是什麼 ?
Kubernetes 提供了**抽象的 Cluster (virtual cluster) 的概念**，讓我們能根據專案不同、執行團隊不同，或是商業考量，將原本擁有實體資源的單一 Kubernetes Cluster ，劃分成幾個不同的`抽象的 Cluster (virtual cluster)`，也就是 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)。

在 [minikube](https://github.com/kubernetes/minikube) 上透過 `kubectl get` 指令，我們可以看到當前有 `default`, `kube-system` 與 `kube-public` 這三個 namespaces，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-get-namespaces.png?raw=true)

- **default**  
  預設的 Namespaces 名稱為 `default`，過去我們產生的物件像是， [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)， [Services](https://kubernetes.io/docs/concepts/services-networking/service/) 等若沒特別指定 Namespace 都是存放在名稱為 **default** 的 namespaces 中。
  
- **kube-system**
  在 Kubernetes 中，較特別的資源都會存放在 `kube-system` 這個 namespace。若是用 `kubectl get all -n kube-system` 查看，可以發現先前介紹的 [kube-dns](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day26) 或是 [heapster](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day23) 都是存放在該 namepsace 中。
  
- **kube-public**  
  `kube-public` 也是個特殊的 namespace，存放在裡面的物件可被所有的使用者讀取。

## [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 有以下幾個特點

- 在同一個 Kubernetes Cluster 中，每個 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 的名稱都是要**獨特的**
- 當一個 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 被刪除時，在該 Namespace 裡的所有物件也會被刪除
- 可以透過 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 限制一個 Namespaces 所可以存取的資源


## Create a new [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

首先，我們要先建立一個名稱為`newspace`的 namespace，指令如下，

```
$ kubectl create namespace newspace
namespace "newspace" created
```

透過 `kubectl get namespaces` 可以查看剛創建好的 namespaces

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-get-namespaces-2.png?raw=true)

## 切換預設 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
若要查看目前在哪個 Namespace 底下，可用以下指令

```
$ kubectl config view | grep namespace:
```


![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-config-default-namespaces-2.png?raw=true)

可以看到預設為 `default`。可以透過 `kubectl config set-context` 指令，將預設的指令切換為 `newspace`，指令如下：

```
$ kubectl config set-context \
> $(kubectl config current-context) \
> --namespace=newspace
Context "minikube" modified.
```

若再用指令查看目前預設的 namespace，可以發現原本的 **default** 變為 **newspace** 了，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-config-default-namespaces-3.png?raw=true)

## 刪除單一 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

透過 `kubectl delete` 指令，可以將我們剛建立好的 `newspace` 刪除，指令如下，

```
$ kubectl delete namespaces newspace
namespace "newspace" deleted
```

需留意的是，**default** 與 **kube-system** 是無法被刪除的 namespaces

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-delete-default-namespace-error.png?raw=true)


# 實作：限制某一 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 的運算資源

以 [hellospace.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/demo-namespaces/hellospace.yaml) 為例，內容如下：

```
apiVersion: v1
kind: Namespace
metadata:
  name: hellospace
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quotas
  namespace: hellospace
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "1"
    requests.memory: 10Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quotas
  namespace: hellospace
spec:
  hard:
    services: "2"
    services.loadbalancers: "1"
    secrets: "1"
    configmaps: "1"
    replicationcontrollers: "10"
```

在 [hellospace.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/demo-namespaces/hellospace.yaml) 中，我們新創建了一個 `hellospace` 的 namespace，且限制該 namespace 

- **運算資源(compute-quotas)**  
  CPU 最多只有 1 core ，以及 memory 的使用被限制在 10Gi 以下
  
- **物件資源(object-quotas)**  
  限制 `hellospace` 最多只能有 2 個 [services](https://kubernetes.io/docs/concepts/services-networking/service/) 物件，且只能有 1 個 loadbalancer, secret, 以及 configmap。

使用 `kubectl create` 創建，

```
$ kubectl create -f ./hellospace.yaml
namespace "hellospace" created
resourcequota "compute-quotas" created
resourcequota "object-quotas" created
```

創建完之後，可用 `kubectl get` 查看在 `hellospace` 裡的 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

```
$ kubectl get resourcequotas -n hellospace
NAME             AGE
compute-quotas   5m
object-quotas    5m
```

用 `kubectl describe` 分別查看 **compute-quotas** 與 **object-quotas** 的內容，

- **compute-quotas**  
![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-describel-computer-quotas.png?raw=true)


- **object-quotas**  
![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-describel-object-quotas.png?raw=true)

可以發現，在 `hellospace` 中已有一個 **secret** 物件，若這時我們再創建一個新的 secret 物件，會發生什麼事呢，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day27/kubectl-cannot-create-secret.png?raw=true)

會看到 Kubernetes 發出 `超出可允許的使用quota` 警告。**代表 hellospace 受 comput-quotas 與 object-quotas 的限制。**

若有興趣的讀者，不妨多做其他操作看看當超出 Resource Quotas 唷。

# 結論
在實際場景中，[Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 與 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 的設計使得 Kubernetes 在管理與處理不同專案的可用性大幅提升。如果對於 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 或 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 還想更深入研究的讀者，不妨參考 [Kubernetes 官方文件](https://kubernetes.io/docs/concepts/policy/resource-quotas/)，會對這兩著的運用更加清楚。

# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
