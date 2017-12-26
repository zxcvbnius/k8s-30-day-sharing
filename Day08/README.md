# [Day 8] 還在用Replication Controller嗎？不妨考慮Deployment





# 前言
在前一天我們 [介紹到Replication Controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07) 。如果讀者看過 [Replication Controller官方文件](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) ，可以看到官方在文件一開頭就表示：

> NOTE: A Deployment that configures a ReplicaSet is now the recommended way to set up replication.

正如昨天所提，雖然 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 雖然看似幫我們解決Pod scaling的問題，然而在實際場景上的應用是不夠的。


### 系統升級(Rollout) & 回滾(Rollback)

若是回溯 [DevOps](https://zh.wikipedia.org/wiki/DevOps) 的歷史，可以發現 [DevOps](https://zh.wikipedia.org/wiki/DevOps) 與 [Agile敏捷開發](https://zh.wikipedia.org/wiki/%E6%95%8F%E6%8D%B7%E8%BD%AF%E4%BB%B6%E5%BC%80%E5%8F%91) 可以說是密不可分，最早 [DevOps](https://zh.wikipedia.org/wiki/DevOps) 一詞也是從 Agile社群中被提出，如果讀者對於這段歷史有興趣，不妨看看 Youtuber 上的一段影片 [The History of DevOps](https://www.youtube.com/watch?v=o7-IuYS0iSE)，會更了解DevOps與Agile之間的關係。而在 Agile 社群中其中一個精神便是`如何正確收集到使用者需求以及快速的回應`，`頻繁的系統升級(Rollout)`更是不可避免的。而對於運維人員，如何在頻繁更新一個服務時做到每次都能`zero downtime(無停機服務遷移)`更是一大挑戰。幸好，這些 Kubernetes 都幫我們做好了。

Kubernetes 官方提供我們 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 元件，不只能幫我們做到Pod scaling，對於	應用服務的rollout與rollback的支援也非常豐富。今天學習筆記內容如下：


 - 認識 Replication Set
 - 介紹 Deployment
 - 實作：透過 kubectl 操作 Deployment 物件 & 常用指令


> 小提醒：今天的程式碼都可以在 [demo-deployment](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day08/demo-deployment) 上找到。






# Replica Set
[Replica Sets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 可以說是進化版的 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)，與 Replication Controller最大的差異在於，[Replica Sets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 提供了`更彈性的selector`。

以 [my-replica-sets.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/demo-deployment/my-replica-sets.yaml) 為例

![kubectl-describe-rs](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/kubectl-describe-rs.png?raw=true)

不同於 Replication Controller 的 selector，只能用`等於`的符號表示，[Replica Set](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 的 selector 支援更多複雜的條件過濾。

 - **apiVersion**  
   如果**kubectl**的版本 >= 1.9，則需使用`app/v1`；如果版本號是在1.9以前的話，則需使用`apps/v1beta2`
 - **spec.selector.matchLabels**  
   在 Replica Set的selector裡面提供了`matchLabels`，matchLabels的用法代表著`等於(equivalent)`，代表Pod的labels必須與`matchLabels`中指定的值相同，才算符合條件。
 - **spec.selector.matchExpressions**
   而`matchExpressions`的用法較為彈性，每一筆條件主要由三個部分組成`key`, `operator`，`value`。以 [my-replica-sets.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/demo-deployment/my-replica-sets.yaml) 中敘述為例，我們指定Pod的條件為 1) env必須為dev 2) env不能為prod。而目前`operator`支援4種條件`In`, `NotIn`, `Exists`, 以及 `DoesNotExis`，更多關於`matchExpressions`的運用可以參考 [官方文件](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)


Replica Set 與昨天提到的Replication Controller 的kubectl指令相似，可以參考 [昨日Replication Controller學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07)

而在 [Kubernetes官方文件](https://kubernetes.io) 中也提到，雖然Replica Set提供更彈性的selector，並不推薦開發者直接使用`kubectl create`等指令創建Replica Set 物件，而是透過 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 來創建新的 Replica Set。





# 介紹 Deployment
[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 可以幫我們達成以下幾件事情：

 - 部署一個應用服務(application)
 - 協助 applications 升級到某個特定版本
 - 服務升級過程中做到無停機服務遷移(zero downtime deployment)
 - 可以Rollback到先前版本

以下，是我們今天會使用到的 [my-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/demo-deployment/my-deployment.yaml)

```
apiVersion: apps/v1beta2 # for kubectl versions >= 1.9.0 use apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-deployment
  template:
    metadata:
      labels:
        app: my-deployment:latest
    spec:
      containers:
      - name: my-pod
        image: zxcvbnius/docker-demo
        ports:
        - containerPort: 3000
```


Deployment [yaml](https://zh.wikipedia.org/wiki/YAML) 檔的寫法與 [Replica Sets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 相似，如果**kubectl**的版本 >= 1.9，則需使用`app/v1`；如果版本號是在1.9以前的話，則需使用`apps/v1beta2`，可以用`kubectl version`查看目前的版本號，

![kubectl-version](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/kubectl-version.png?raw=true)

> 由於筆者目前仍使用`1.8`的版本，所以apiVersion仍需使用`apps/v1beta2`。

接著我們可以使用`kubectl create`指令來新建一個 Deployment 物件，

```
$ kubectl create -f ./my-deployment.yaml
deployment "hello-deployment" created
```

用`kubectl get`查看deployment與Pod的狀態，

![kubectl-get-deployment](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/kubectl-get-deployment.png?raw=true)

可以發現 Deployment已自動幫我們創建Pod，且這個Pod都帶有**app=my-deployment**的label，而在同時，Deployment也會自動幫我們建立一個`Replication Set`來管理這些Pod

```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
hello-deployment-6577d8cc46   3         3         3         22m
```





# 實作：透過 kubectl 操作 Deployment 物件


在開始實作之件，想先分享幾個常見的指令：

| Deployment相關指令 |                           指令功能                     |
| ----------------- | ---------------------------------------------------- |
| kubectl get deployments | 取得目前Kubernetes中的deployments的資訊 |
| kubectl get rs | 取得目前Kubernetes中的Replication Set的資訊 |
| kubectl describe deploy  &lt;deployment-name> | 取得特定deployment的詳細資料 |
| kubectl set image deploy/ &lt;deployment-name>  &lt;pod-name>: &lt;image-path>: &lt;version> | 將deployment管理的pod升級到特定image版本 |
| kubectl edit deploy  &lt;deployment-name> | 編輯特定deployment物件 |
| kubectl rollout status deploy  &lt;deployment-name> | 查詢目前某deployment升級狀況 |
| kubectl rollout history deploy  &lt;deployment-name> | 查詢目前某deployment升級的歷史紀錄 |
| kubectl rollout undo deploy  &lt;deployment-name> | 回滾Pod到先前一個版本 |
| kubectl rollout undo deploy  &lt;deployment-name> --to-revision=n | 回滾Pod到某個特定版本 |


接下來，將會根據上面幾個常用指令，在我們本機端的環境中進行實際操作。


#### 取得Deployment/Replication Set/Pod 基本資訊
相信眼尖的讀者發現，若是要取得在Kuberntes的物件資訓，都是透過`kubectl get`指令。如果想要看Kubernetes Cluster中所有目前已被建立的物件，可以使用`kubectl get all`一次取得所有資訊。

![kubectl-get-all](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/kubectl-get-all.png?raw=true)


若是對於指令還不太熟悉，也可以在終端機輸入`kubectl get`，如此便能得到Kubernetes`目前所有提供的元件型別`以及`每個元件型別支援的縮寫格式`。


```
$ kubectl get
You must specify the type of resource to get. Valid resource types include:

    * all
    * certificatesigningrequests (aka 'csr')
    * clusterrolebindings
    ...
    * pods (aka 'po')
    ...
```

為了實作如何`利用Deployment 升級web app`，筆者預先將[先前 demo 用的 web application](https://hub.docker.com/r/zxcvbnius/docker-demo/tags/) v2版本上傳到 [Docker Hub](https://hub.docker.com)，在`v2`的版本中，收到request之後會回傳`Hello World! v2`


![docker-demo-v2](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day08/docker-demo-v2.png?raw=true)


而在上面，我們使用`kubectl create`創建好一個Deployment物件之後，我們可以先創建一個 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 來測試是否我們創建的web app有正常運作。

```
$ kubectl expose deploy hello-deployment --type=NodePort --name=my-deployment-service
service "my-deployment-service" exposed
```
創建好之後，我們可以透過`minikube server`取得目前`my-deployment-service`的網址，

```
$ minikube service my-deployment-service --url
http://192.168.99.100:30390
```

接著我們可以在本機端的瀏覽器訪問 [http://192.168.99.100:30390](http://192.168.99.100:30390)


![deployment-demo1](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day08/deployment-demo1.png)

可以看到web app正常運作中。接著我們可以透過`docker set image`來rollout我們的 web app

```
$ kubectl set image deploy/hello-deployment my-pod=zxcvbnius/docker-demo:v2.0.0
deployment "hello-deployment" image updated
```

這時我們也可以使用`kubectl rollout status`來查詢目前升級的狀態

```
$ kubectl rollout status deploy hello-deployment
deployment "hello-deployment" successfully rolled out
```

可以看到 deployment **hello-deployment** 已成功升級完成。

如果再重新訪問一次 [http://192.168.99.100:30390](http://192.168.99.100:30390)，可以看到瀏覽器的畫面變成`Hello World! v2`了！


![deployment-demo2](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day08/deployment-demo2.png)


如果在升級的過程中，查看`kubectl get pod`，會發現Deployment在升級web app時，並不會把原本的pod直接砍掉，而是會另外生成Pod來取代原本Pod，來達到無痛升級的需求(zero downtime)。也就是說，在我們的Kubernetes Cluster中同時間會存在6個Pod。

![kubectl-rollup-pod-status](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day08/kubectl-rollup-pod-status.png)


如果不想要同時間存在這麼多Pod，我們可以在Deployment的設定檔中指定**strategy.rollingUpdate.maxSurge**的數量。在今日學習筆記中後段，會介紹如何設定`maxSurge`。


除了透過`kubectl set image`更新Pod的image之外，我們也可以透過`kubectl edit`來進行更新，輸入以下指令

```
$ kubectl edit deploy hello-deployment
```

![kubectl-edit-deploy](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day08/kubectl-edit-deploy.png)

除了可以看到原先的我們設定的欄位以外，也會發現多了幾個不一樣的欄位。

 - **strategy**  
   Kubernetes為了讓我們可以確保rollout的時候，按照我們所想的方式。所以在Deployment Configuration中，有一個**strategy**的設定值，可以將我們希望行為放在裡面。

 - **strategy.rollingUpdate.maxSurge**   
   maxSurge可以是`百分比數`，也可以是`數字`。代表在rollout的過程中，最多可以比原先指定的Pod數量多出多少。好比`原本replicas為3`，`maxSurge為1`，代表在升級的過程中，deployment頂多會多產生一個Pod。

 - **strategy.rollingUpdate.maxUnavailable**   
   代表在升級過程中，可以容忍多少Pod無法使用，如果`maxSurge`的數字非0的話，`maxUnavailable`的數字也不能為0。

最後，我們可以透過`kubectl rollout history`來查該Deployment看過去rollout的紀錄

![kubectl-rollout-history](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day08/kubectl-rollout-history.png)

可以看到過去`hello-deployment`總共經歷2次的rollout。一次是我們第一次創建Deployment的時候，一次是我們升級成`v2`版本的時候。而`CHANG-CAUSE`可以紀錄我們每次rollout的指令。例如，我們希望將Pod“升級”到latest版本。

```
$ kubectl set image deploy/hello-deployment my-pod=zxcvbnius/docker-demo --record
deployment "hello-deployment" image updated
```

![kubectl-rollout-history-record](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day08/kubectl-rollout-history-record.png)

可以看到`CHANGE-CAUSE`多了紀錄，之後如果我們想要`revert`到特定版本，也可以根據`CHANGE-CAUSE`來決定。

如果我們想要把目前版本`rollback`到上一版，我們可以使用`kubectl rollout undo`

```
$ kubectl rollout undo deployment hello-deployment
deployment "hello-deployment" rolled back
```

如果這時再去檢查`kubectl rollout history`，會發現多了一個版本

```
$ kubectl rollout history deploy hello-deployment
deployments "hello-deployment"
REVISION  CHANGE-CAUSE
1         <none>
3         kubectl set image deploy/hello-deployment my-pod=zxcvbnius/docker-demo --record=true
4         <none>
```

最後，要介紹的是如何rollback回到特定版本，已回滾到`REVISION 3`為例，可以使用以下指令

```
$ kubectl rollout undo deploy hello-deployment --to-revision=3
```

在rollback成功之後，若是再次查詢`kubectl rollout history`會發現又多了一個`REVISION`囉。


# 總結
以上，是今天學習筆記中想跟大家分享 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 的部分。雖然今天的學習筆記無法分享Deployment的其他用途，但光是`rollup`與`rollback`相信讀者也可以感受到Deployment的強大。而在明天的學習筆記中，將會分享我們在過去幾天中常提及的Kubernetes的元件 - [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 。

> 如果對於A/B test有興趣的朋友，可以參考官網提供的 [Canary Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#canary-deployment) ，相信會有不少的收穫。


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )




# 參考連結

 - [Kubernetes官網](https://kubernetes.io)
 - [Scalability Wikipedia](https://en.wikipedia.org/wiki/Scalability#Horizontal_and_vertical_scaling)
