# [Day 7] 如何擴展我的pods?! - Replication Controller




# 前言
在前兩天認識 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 以及 [Node元件](https://kubernetes.io/docs/concepts/architecture/nodes) 之後，今天學習筆記會分享如何利用 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 來擴展(Scaling)以及管理 Pod。今天學習筆記內容如下：

 - 介紹 Stateful App 與 Stateless App
 - 什麼是 Replication Controller
 - 實作：透過 kubectl 操作 Replication Controller 物件


> 小提醒：今天的程式碼都可以在 [demo-replication-controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07/demo-replication-controller) 上找到唷





# 什麼是 Stateless ? 什麼是 Stateful ?
在開始認識 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 之前，筆者想先介紹 **Stateless** 與 **Stateful**。

[Stateless](https://softwareengineering.stackexchange.com/questions/346867/how-to-keep-applications-stateless) 是指這個應用服務或功能，不因時間、是否有資料寫入、接收request的次數，或是任何狀態的改變影響服務回傳的資料。而`web applications`都應該盡可能是 **stateless** 的，除非有很好的理由需要在app中保存這些`狀態`。

以下述function為例，如果有個function是兩將外部傳進來的兩個值相加並回傳，不受其他因素影響回傳值，我們可以稱該function為`stateless function`

```
function int sum(int a, int b) {
    return a + b;
}
```

而與 [Stateless](https://softwareengineering.stackexchange.com/questions/346867/how-to-keep-applications-stateless) 相對的，便是`Stateful Application`。過去**傳統關聯性資料庫(traditional database)**，像是[MySQL](https://zh.wikipedia.org/zh-tw/MySQL)，[PostgreSQL](https://www.postgresql.org/) 等都是**Stateful Application**。會紀錄目前每筆資料的狀態，即便關掉服務再次重啟，這些資料仍應被保存著。

以下述function為例，當我們有個function 可以用來記錄被呼叫次數時，我們就可以稱做為`stateful function`

```
int count = 0;
function int counter() {
	count++;
	return count;
}
```

如果還有印象 [container的幾項特性](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03) ，會記得`container`並不會保存資料，每個`container`的內部資料都會因為`container`的消失而消失。所以我們需要外部的資源去儲存這些資料，在日後的[學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day01) 中會有更詳細的介紹。在今天學習筆記中，將會藉由[我們之前打造 stateless 的 web application](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03)，與大家分享的是container如何透過 Replication Controller實現 containers 的 `橫向擴展(Horizontal scaling)`。  



> Scaling又可以分為兩種， [Horizontal scaling]() 與 [Vertical scaling]() 。`橫向擴展(Horizontal scaling)`代表，我們可以透過**增加更多的機器節點**，獲取更多資源，來分擔原有的工作內容。而`縱向擴展(Vertical scaling)`則表示我們可以在單一節點上，新增更多的 **CPU**，**RAM** 來獲得更多運作資源。如果讀者對scaling想更深入了解，不妨參考 [連結](https://en.wikipedia.org/wiki/Scalability#Horizontal_and_vertical_scaling)




# 什麼是 Replication Controller
[Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 就是Kubernetes上用來管理Pod的數量以及狀態的controller，我們可以用 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 在Kubernetes上做到以下幾件事：

 - 每個Replication Controller都有屬於自己的 [yaml](https://zh.wikipedia.org/wiki/YAML) 檔
 - 在Replication Controller設定檔中`可以指定同時有多少個相同的Pods`運行在Kubernetes Cluster上
 - 當某一Pod發生crash, failed，而終止運行時，Replication Controller會幫我們自動偵測，並且自動創建一個新的Pod，確保`Pod運行的數量與設定檔的指定的數量相同`
 - 當機器重新開啟時，之前在機器上運行的 Replication Controller 會自動被建立，確保pod隨時都在運行。

以下，是我們今天會使用到的 [my-replication-controller.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/demo-replication-controller/my-replication-controller.yaml)


```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-replication-controller
spec:
  replicas: 3
  selector:
    app: hello-pod-v1
  template:
    metadata:
      labels:
        app: hello-pod-v1
    spec:
      containers:
      - name: my-pod
        image: zxcvbnius/docker-demo
        ports:
        - containerPort: 3000
```

由於 `apiVersion`，`metadata`在介紹 [Pod](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05) 時已介紹過，在這裡不再撰述，不過可以看到Replication Controller中的`spec`與 [Pod的yaml檔](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/demo-pod/my-first-pod.yaml) 有些不相同。


 - **spec.replicas & spec.selector**  
   在`spec.replicas`中，我們必須定義`Pod的數量`，以及在`spec.selector`中指定我們要選擇的Pod的條件(labels)。

 - **spec.template**   
   在spec.template中我們會去定義pod的資訊，包含Pod的labels以及Pod中要運行的container。

 - **spec.template.metadata**      
   則是Pod的labels，metadata.labels必須被包含在`select`中，否則在創建Replication Controller物件時，會發生error。

 - **spec.template.spec**    
   最後spec的部分則是定義container，可以參考 [Pod的yaml檔](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/demo-pod/my-first-pod.yaml) 的範例，在我們的範例中，一個Pod只有一個container。


總結以上，可以知道在 [my-replication-controller.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/demo-replication-controller/my-replication-controller.yaml) 中我們想創建一個Replication Controller物件，來`確保擁有"app=hello-pod-v1"label的Pod，在任何時候，Kubernetes Cluster中的數量都為3`。




# 實作：透過 kubectl 操作 Replication Controller 物件
在建立Replication Controller之前，我們先檢查一下`minikube`狀態是否`Ready`

```
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    2d        v1.8.0
```

確認沒問題之後，可以透過`kubectl create`的指令，創建一個新的Replication Controller物件，

![kubectl-create-rc](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/kubectl-create-rc.png?raw=true)

當新增replication controller成功之後，我們可以使用`kubectl get rc`查看目前狀態

![kubectl-get-rc](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/kubectl-get-rc.png?raw=true)

從圖中可以知道，`my-replication-controller`這個物件目前管理3個Pod，且3個Pod的狀態皆為`Ready`，這時我們在查看一下Pod的狀態

![kubectl-get-pod](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/kubectl-get-pod.png?raw=true)

可以發現在Kubernetes Cluster中，replication controller自動幫我們創建好三個pod，且這三個pod是名稱是unique的。如果我們用`kubectl describe`指令去查看其中一個pod的詳細資料

```
$ kubectl describe pod my-replication-controller-4ftnj
Name:           my-replication-controller-4ftnj
Namespace:      default
Node:           minikube/192.168.99.100
.....
Created By:     ReplicationController/my-replication-controller
Controlled By:  ReplicationController/my-replication-controller
```

可以看到這個pod是由`my-replication-controller`物件創建的。如果這時我們手動刪除其中一個Pod，我們來看看會發生什麼事情，

```
$ kubectl delete pod my-replication-controller-4ftnj
pod "my-replication-controller-4ftnj" deleted
```

![kubectl-rc-recreate-new-pod](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/kubectl-rc-recreate-new-pod-1.png?raw=true)

可以看到replication controller偵測到一個Pod終止服務時，產生另外一個新的Pod，來確保Pod的數量。

我們也可以透過`kubectl scale`來scaling Pod的數量，指令如下

```
$ kubectl scale --replicas=4 -f ./my-replication-controller.yaml
replicationcontroller "my-replication-controller" scaled
```

這時候再去查看`kubectl get rc`或是`kubectl get pods`，可以發現Pod的數量多一個了。我們也可以用同樣的方式去縮減Pod的數量。

```
$ kubectl get rc
NAME                        DESIRED   CURRENT   READY     AGE
my-replication-controller   4         4         4         26m
```

也可以透過`kubectl describe`去看目前replication controller的詳細資料，更多log資訊可參考 [kubectl-describe-rc.log](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/demo-replication-controller/kubectl-describe-rc.log)

```
$ kubectl describe rc my-replication-controller
Name:         my-replication-controller
Namespace:    default
Selector:     app=hello-pod-v1
Labels:       app=hello-pod-v1
....
```


最後，當我們`刪除replication controller`時，要特別注意，由replication controller產生的`pod`也會因此而終止服務。

![kubectl-delete-rc](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day07/kubectl-delete-rc.png?raw=true)


如果你希望刪掉replication controller之後，這些Pod仍然運行，可以指定**--cascade=false**，指令如下：

```
$ kubectl delete rc my-replication-controller --cascade=false
```


如果Pod的數量很大，將會花一些時間來完成整個刪除動作，可以使用指令強迫pod立即結束。

```
$ kubectl delete pods <pod> --grace-period=0 --force
```

更多關於`kubectl delete`的指令應用，可以參考[Kubernetes官網文件](https://kubernetes-v1-4.github.io/docs/user-guide/kubectl/kubectl_delete/)，裡面有詳細介紹，有興趣的讀者不妨看一看。




# 總結
希望今天介紹完 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) ，讀者能對Kubernetes有多一層認識，雖然replication controller看似能幫我們解決很多問題，但在實務上，應用服務(application)常會遇到rollout以及rollback的情形，若只是使用 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 免不了需要許多手動的部分。而明天學習筆記中將會介紹 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) ， [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 不只減少需要我們手動操作的部分，甚至讓應用服務rollout與rollback也變得簡單許多。


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )




# 參考連結

 - [Kubernetes官網](https://kubernetes.io)
 - [Scalability Wikipedia](https://en.wikipedia.org/wiki/Scalability#Horizontal_and_vertical_scaling)
