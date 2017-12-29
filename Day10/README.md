# [Day 10] Kubernetes世界不可缺少的 - Labels





# 前言
當 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 數量越來越多時，管理的維度也會逐漸複雜，譬如如何將 Pod 在不同地區上部署，不同層級的 Pod 如何分開管理，如何決定哪個類型的 Pod，部署在相對應的 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 上，都是我們在實際場景上需要考量的部分。而 Kubernetes 提供了我們 [Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 這個元件，讓我們能對每個不同屬性的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 都能貼上屬於他們自己的標籤、且將有不同種類的標籤 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 做分群管理。今天的學習筆記如下：

 - Labels 是什麼 ？
 - Annotations 是什麼 ？
 - 實作：如何將 Pod 部署到特定的 Node 上





# Labels 是什麼 ？
簡單來說，Labels就是**一對具有辨識度的key/value**。以下面為例：

 - "release" : "stable"，"release" : "qa"
 - "enviroment": "dev"，"enviroment": "production"
 - "tier": "backend", "tier": "frontend"
 - "department": "enginnerting", "department": "marketing", "department": "finance"

若Kubernetes Cluster中的 Pod 物件帶有標籤，由上面幾個 labels 我們可以清楚知道，他們的功用是什麼、層級是什麼，他們服務的對象是哪些部門的人。[Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 有以下幾個特點：

 - 每個物件可以同時擁有許多個labels(multiple labels)
 - 可以透過 [Selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)，幫我們縮小要尋找的物件。
 - 目前 API 提供不再只是一個 **key對應一個value(Equality-based requirement)的關係**，我們也可以使用  [matchExpressions](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 來設定更有彈性的`Labels`。




# Annotations 是什麼 ？
然而，如果是**沒有識別用途的標籤**，Kubernetes 也提供了我們一個 [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 元件。以 Pod 為例，我們可以在Pod的 [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 紀錄該 Pod的發行時間，發行版本，聯絡人email等。[Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 主要是方便開發者、以及系統管理者管理上的方便，不會直接被 Kubernetes使用。

以下面 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/demo-labels/my-pod.yaml) 為例


```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: webserver
    tier: backend   
  annotations:
    version: latest
    release_date: 2017/12/28
    contact: zxcvbnius@gmail.com
spec:
  containers:
  - name: pod-demo
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000        
```

透過`kubectl create`來創建一個新 Pod 物件，

```
$ kubectl create -f ./my-pod.yaml
pod "my-pod" created
```

接著我們可以透過`kubectl get pods` 來取得目前 Pod 的資訊，若是在指令後面加上`--show-labels`則也會顯示 Pod的labels，

![kubectl-get-pod-with-labels](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-get-pod-with-labels.png?raw=true)  

可以看到我們剛剛指定的`labels`。我們也可以透過 `kubectl describe` 去查看該Pod 的 annotations 有哪些

```
$ kubectl describe pods my-pod
```

![kubectl-describe-with-annotations](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-describe-with-annotations.png?raw=true)


### 動態新增 Labels
除了在定義檔一開始就加入 Labels 以外，我們也可以透過 `kubectl label`的指令，來為我們的 Pod 新增標籤，指令如下：

```
$ kubectl label pods my-pod env=production
pod "my-pod" labeled

$ kubectl get pods my-pod --show-labels
NAME      READY     STATUS    RESTARTS   AGE       LABELS
my-pod    1/1       Running   0          1h        app=webserver,env=production,tier=backend
```
若是再次查看 `my-pod` 的 Labels，就可以看到`env=production`被新增在 `LABELS`中。





# 實作：如何將 Pod 部署到特定的 Node 上
同樣我們也能將 [Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 的觀念套用在 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 上。可以透過幫每個 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 貼標籤讓 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 運行在特定的 Node 上。以下我們將做兩件事情：

 - 在 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 上貼上標籤
 - 在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 新增 `nodeSelector` 定義


> 雖然目前只在本機端(single node)上實作，但透過以下的實作，在將來操作多個 Nodes 時，也會更有感覺。


### 在 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 上貼上標籤
在新增label前，先檢查一下目前 Node 有的labels，

![kubectl-get-node-with-labels](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-get-node-with-labels.png?raw=true)

然後我們對 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/demo-labels/my-pod-with-node-selector.yaml) 做一些修改，

![kubectl-add-node-selector](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-add-node-selector.png?raw=true)


創建一個新的`my-pod`物件  

```
$ kubectl create -f ./my-pod-with-node-selector.yaml
pod "my-pod" created
```

成功之後，我們可以查看目前 Pod 的狀態，會發現：

![kubectl-pod-pending-status](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-pod-pending-status.png?raw=true)

會發現目前`my-pod`的狀態一直在`Pending`的狀態，如果在用`kubectl describe`仔細查看會發現錯誤的原因是因為沒有找到`符合的 Node `部署

![kubectl-describe-pod-with-wrong-node-selector](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-describe-pod-with-wrong-node-selector.png?raw=true)


如果我們動態新增一個`label`到目前的 `minikube node` 上，

```
$ kubectl label node minikube hardware=high-memory
node "minikube" labeled

$ kubectl get node --show-labels
NAME       STATUS    ROLES     AGE       VERSION   LABELS
minikube   Ready     <none>    6d        v1.8.0    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,hardware=high-memory,kubernetes.io/hostname=minikube
```

可以發現`minikube`上多了一個 label ，這時若是我們再去查看 my-pod 的狀態會發現

```
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
my-pod    1/1       Running   0          7m
```

已成功運作，若是我們再次用`kubectl describe` 查看 `my-pod` 的狀態，可以看到更詳細的部署過程

![kubectl-pod-log-history](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day10/kubectl-pod-log-history.png?raw=true)


> 讀者可以試著設想，若是有兩種不同類型的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) ，一個需要高量的memory，另外一個是需要高量的CPU。透過 `nodeSelector` 與 `labels`，我們可以將這些不同種類型的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 部署在不同類型的 Node，讓資源能更有效被利用。


# 總結
除了透過 [NodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 來指定特定的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 到特定的 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 之外，我們也可以透過 [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 來幫我們指定。雖然 [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 還是`beta`版本，但它提供了更多更彈性的功能！在未來的學習筆記中若有機會，也會與大家分享。


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )





# 參考連結

 - [Kubernetes官網](https://kubernetes.io)
