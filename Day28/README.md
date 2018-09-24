# 前言
如果有運維服務的經驗，可能多少都有遇過 Server maintenance。以 [Linode](https://www.linode.com/) 這家雲端服務供應商而言，他們的價格比起 [AWS](https://aws.amazon.com) 或 [GCP](https://cloud.google.com/?hl=zh-tw) 便宜不少，然而若有在 [Linode](https://www.linode.com/) 租借過  server 的讀者，應該也常常會收到 Server 需要維護的通知。這時候運維人員可能會通知內部使用這台 server 的開發團隊，要求將部署在該台 server 上的 application，部署在另外一台。而這樣一來一往的溝通可能就會消耗了好幾個工作天。Kubernetes 提供了我們一個好用的指令，讓運維人員可以直接將某台 Node 上的 application 搬移到另外一台 Node 上。今天學習筆記如下，

- 介紹什麼是 [Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller)
- 實作：在 [minikube](https://github.com/kubernetes/minikube) 上實現 Node 的隔離與恢復

# 什麼是 [Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller) ?
在開始介紹怎麼隔離與恢復 Node 之前，想先介紹 [Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller)。[Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller) 是 Kubernetes 中用來管理 Node 的一個物件，它可以做到以下幾件事情，
- 知道目前 Kubernetes 中可用的 Node 清單 (Available Node List)
- 定期監控每個 Node 的狀態，若是有 Node 呈現 **unhealthy** 的狀態時，就會將該 Node 從清單中移除，而在該 Node 上的 Pod 就會被重新分配到其他 Node 上。
- 當 Node 的狀態狀態變回 **healthy** 時，[Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller) 則會把該 Node 加回可用的清單中。

所以當我們想將某個 Node 從可用清單移除時，[Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller) 也會幫我們處理。避免 Master Node 再將資源創建在該 Node 上，已以下範例為例。

## 隔離 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 

如果我們要從 Kubernetes Cluster 移除一個 node 只需要下一個指令

```
$ kubectl drain {node_name}
```


`kubectl drain` 代表將該 Node 狀態變更為維護模式，如此在該 Node 上面的 Pod，就會轉移到其他 Node 上。然而值得注意的是，若不是透過 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 或是 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 等創建好的 Pod，則該指令會失敗，除非加上 **--force**

```
$ kubectl drain {node_name} --force
```

> 這也是為什麼 Kubernetes 不建議直接創建 Pod ，而是推存透過 Controller 來管理 Pod 的原因。


## 恢復 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 

在 Kubernetes 中，將一個 Node 加回 Kubernetes 的指令也非常簡單，內容如下，

```
$ kubectl uncordon {node_name} 
```

即可把原先隔離的 Node 重新加回 Kubernetes 中

# 實作：在 [minikube](https://github.com/kubernetes/minikube) 上實現 Node 的隔離與恢復

以 [minikube](https://github.com/kubernetes/minikube) 為例，首先我們先透過 [hello-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day28/demo-nodes/helloworld-depolyment.yaml) 設定檔建立 `hello-deployment` 物件，

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3
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


用 `kubectl create` 創建，

```
$ kubectl create -f ./helloworld-depolyment.yaml
deployment "helloworld-deployment" created
```

用 `kubectl get` 查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day28/kubectl-get-all.png?raw=true)

可以看到目前 Pod 的狀態為 `RUNNING` ，接著我們透過 `kubectl get nodes` 查看

```
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    17h       v1.8.0
```

目前 minikube 的狀態也為 `Ready`，這時我們透過 `kubectl drain` 將 minikube 標示隔離狀態，來看看 minikube 的狀態與上面的 Pod 的狀態會如何，

```
$ kubectl drain minikube --force
...
node "minikube" drained
```

若再次去查看目前狀態，可以發現，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day28/kubectl-get-all-2.png?raw=true)

由於我們只有一個 Node，因此 Pod 的狀態皆為 `Pending`，而 minikube 的狀態也變為 `SchedulingDisabled`，代表當 Kubernetes Cluster 若是再接收到創建物件的指令後，也不會把物件分配給該 Node。若我們希望 minikube 能再回到排程中，可以透過 `kubectl uncordon` 實現，

```
$ kubectl uncordon minikube
node "minikube" uncordoned
```

再次查看 Pod 與 Node 的狀態，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day28/kubectl-get-all-3.png?raw=true)

可以看到 Pod 又都重新運行了，如果想知道每個 Pod 是運行在哪個 Node 上，可以加上 `-o wide`，指令如下

```
$ kubectl get deploy,pods,nodes -o wide
```


![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day28/kubectl-get-all-4.png?raw=true)

# 總結
Kubernetes 提供了我們兩個好用的指令，**kubectl drain** 與 **kubectl uncordon** 讓運維者可以直接在 Kubernetes Cluster 暫時移除需要維護的機器，以及結束維護之後再將該機器重新加入 Cluster 中。開發者也無需重新將應用程式部署在新的機器上，因為 Kubernetes 都幫我們做好了。最後兩天的學習筆記中，將介紹 Kubernetes 中最核心的角色 Master Node ，讓我們更加清楚Kubernetes 內部到底是如何運作的。


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
