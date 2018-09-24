# [Day 5] 在Minikube上跑起你的docker containers - Pod


# 前言
回顧前幾天的學習筆記，我們現在已經知道

 - [什麼是Kubernetes](https://ithelp.ithome.com.tw/articles/10192401)
 - 如何在[本機端透過minikube架起Kubernetes Cluster](https://ithelp.ithome.com.tw/articles/10192490)
 - 以及，[如何把docker image上傳到docker registry](https://ithelp.ithome.com.tw/articles/10192824)

在前四天打好基礎後，今天的學習筆記中，將會介紹

- 認識Kubernetes最小運行單位 - Pod
- 如何與Pod中的container互動
- 常見的kubectl指令

> 小提醒：今天的程式碼都可以在 [demo-pod](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05/demo-pod) 上找到唷




# 認識Kubernetes最小運行單位 - Pod
在開始將我們 [前天打造的docker container](https://ithelp.ithome.com.tw/articles/10192519) 運行在Kubernetes之前，我們需要先認識`Pod`。


## Pod 是什麼
Kubernetes上會運行很多個不同種類型的應用服務(applications)，而一個 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 在Kubernetes世界中就相當於一個application。  
Pod有以下特點，

 - 每個Pod都有屬於自己的 [yaml](https://zh.wikipedia.org/wiki/YAML) 檔
 - 一個Pod裡面可以包含一個或多個docker container
 - 在同一個Pod裡面的containers，可以用local port numbers來互相溝通

在這裡我們會以昨天打造好的 [docker image](https://hub.docker.com/r/zxcvbnius/docker-demo/) 來做示範，若讀者有自己的Docker Image，在以下的指令中都可以替換成自己的docker image。

> 關於示範的docker image，可以參考 [[Day 3] 打造你的Docker containers](https://ithelp.ithome.com.tw/articles/10192519)


## 如何建立一個 Pod

以下 [my-first-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/demo-pod/my-first-pod.yaml) 是我們今天會用到的yaml檔，

```
# my-first-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: webserver
spec:
  containers:
  - name: pod-demo
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000
```

 - **apiVersion**  
   apiVersion是代表目前Kubernetes中該個元件的版本號。以上述的例子`Pod`是`v1`，而`v1`也是目前Kubernetes中核心的版本號。在日後也會陸續看到`betav1`, `v1alpha1`等版本號，更多Kubernetes API的版本號，可至 [官網 API versioning](https://kubernetes.io/docs/reference/api-overview/#api-versioning)查看。   


 - **metadata**  
   在metadata中，有三個重要的Key，分別是`name`, `labels`, `annotations`。
    - **metadata.name**    
      我們可以在`metadata.name`的欄位指定這個Pod的名稱，這裡Pod的名稱就是my-pod
    - **metadata.labels**          
      而`metadata.labels`Kubernetes的是核心的角色，Kubernetes會透過`Label Selector`將Pod分群管理。在之後 [[Day 10] Kubernetes世界不可缺少的 - Labels]() 章節會詳細介紹Labels的功能。
    - **metadata. annotations**          
      annotations的功能與labels相似，相較於labels，annotations通常是使用者任意自定義的附加資訊，提供外部進行查詢使用，像是版本號，Pod發布日期等等。


 - **spec**  
   最後spec的部分則是定義container，在這個範例中，一個Pod只一個container。
    - **container.name**    
      我們可以在這container中設定container的名稱
    - **container.image**       
      image則是根據 [Docker Registry](https://docs.docker.com/registry/) 提供的可下載路徑。
    - **container.ports**          
      最後ports的部分則是可以指定，該container有哪些port number是允許外部資源存取的，而在這裡我們只允許container中的port 3000對外開放。

在了解 [my-first-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05/demo-pod/my-first-pod.yaml) 每一行在做什麼事情之後，我們可以使用`kubectl create`指令，在Kubernetes cluster中建立Pod物件，

![kubectl-create-pod](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-create-pod.png?raw=true)

這樣，就建立好my-app這個Pod了，可以用`kubectl get pods`查看目前pods的狀態，

![kubectl-pod-creating](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-pod-creating.png?raw=true)

可以看到`ContainerCreating`的狀態，如果再等一下就會看到狀態變為`Running`，代表Pod已經正常開始跑了。

![kubectl-pod-running](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-pod-running.png?raw=true)

我們也可以用`kubectl describe`可以看到更多關於這個`my-pod`的資訊，包含我們設置的Pod Name，labels, 以及這個pod的歷史狀態，更多的log在 [describe-my-pod.log](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-describe-my-pod.log)

```
$ kubectl describe pods my-pod
Name:         my-pod
Namespace:    default
.....
Events:
  Type    Reason                 Age   From               Message
  ----    ------                 ----  ----               -------
  Normal  Scheduled              7m    default-scheduler  Successfully assigned my-pod to minikube
  Normal  SuccessfulMountVolume  7m    kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-wxjzb"
  Normal  Pulling                7m    kubelet, minikube  pulling image "zxcvbnius/docker-demo"
  Normal  Pulled                 7m    kubelet, minikube  Successfully pulled image "zxcvbnius/docker-demo"
  Normal  Created                7m    kubelet, minikube  Created container
  Normal  Started                7m    kubelet, minikube  Started container
```




# 如何與Pod中的container互動
在`my-pod`中，有個container運行API server, port number 3000。然而，我們要怎麼與他做互動呢？常見存取Pod資源的方法有兩種，


### 方法1：透過kubectl port-forward
`kubectl`提供一個指令`port-forward`能將pod中的某個port number，與本機端的port做mapping。在今天最後[常用kubectl指令](#常見的kubectl指令)章節中會提到更多關於 `kubectl port-forward` 指令的介紹，

```
$ kubectl port-forward my-pod 8000:3000
Forwarding from 127.0.0.1:8000 -> 3000
```

看到`Forwarding from 127.0.0.1:8000 -> 3000`代表port-forward成功，現在可以在本機端的瀏覽器打開[http://127.0.0.1:8000](http://127.0.0.1:8000)，就可以看到我們運行的[API Server](https://ithelp.ithome.com.tw/articles/10192519) 吐回來的訊息。

![kubectl-port-forward](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-port-forward.png?raw=true)


### 方法2：建立一個 Service
在之後的學習筆記中也會針對Kubernetes中的[Service](https://kubernetes.io/docs/concepts/services-networking/service/)詳細介紹。而在這章節，我們則是透過`kubectl expose`指令幫我們在Kubernetes建立一個[Service](https://kubernetes.io/docs/concepts/services-networking/service/)物件。  

kubectl port-forward是將pod的port mapping到本機端上，而kubectl expose則是將pod的port與Kubernetes cluster的port number做mapping。

我們先可以使用`minikube status`查看目前minikube-vm是使用本機端哪個內部網址。

![minikube-status](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/minikube-status.png?raw=true)

接著輸入`kubectl expose`指令，創建一個`my-pod-service`的service物件

```
$ kubectl expose pod my-pod --type=NodePort --name=my-pod-service
service "my-pod-service" exposed
```

這時我們可以再輸入`kubectl get services`查看目前運行的services有哪些

```
$ kubectl get services
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP          9h
my-pod-service   NodePort    10.97.118.40   <none>        3000:30427/TCP   1m
```

可以看到`my-pod-service`將`my-pod`的port number 30003與minikube-vm上的port number 30427做mapping。接著可以使用`minukube service`的指令快速找到`my-pod-service`的url

```
$ minikube service my-pod-service --url
http://192.168.99.104:30427
```

這時再從你的本機端上的瀏覽器打開[http://192.168.99.104:30427](http://192.168.99.104:30427)，就可以看到`hello world`的字樣。

![minikube-service-url](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/minikube-service-url.png?raw=true)


> Kubernetes Cluster內部會有一套網路系統，會替每個Pod建立一個cluster ip，這個ip是由Kubernetes內部隨機產生的。這個cluster ip只有Cluster內部資源可以使用；外部資源是無法透過cluster ip與pod互動，所以我們需要再建立一個[Service](https://kubernetes.io/docs/concepts/services-networking/service/)元件讓Cluster以外的服務也可以與Pod做互動。在`kubectl get services`可以看到`TYPE`,`CLUSTER-IP`,`EXTERNAL-IP`，在之後 [[Day 9] 如何讓外部服務與pods的溝通管道 - Services]() 會再詳細介紹。


# 常見的kubectl指令

以下列出幾種我們常用到的指令，

### 取得Kubernetes Cluster中所有正在運行的pods的資訊

```
$ kubectl get pods
```

如果加上`--show-all`，則會顯示所有pods

```
$ kubectl get pods --show-all
```


### 取得某一個Pod的詳細資料

```
$ kubectl describe pod <pod>  
```

### 將某一Pod中指定的port number expose出來讓外部服務存取(建立一個新的service物件)

```
$ kubectl expose pod <pod> --port=<port> --name=<service-name>
```

### 將某一Pod中指定的port number mapping到本機端的某一特定port number

```
$ kubectl port-forward <pod> <external-port>:<pod-port>
```

### 當一個container起來之後，有時希望能進到container內部去看logs，可以使用`kubectl attach`這個指令

```
$ kubectl attach <pod> -i
```


### 可以對pod下一個內部指令

```
$ kubectl exec <pod> -- <command>
```

如果還記得[第三天](https://ithelp.ithome.com.tw/articles/10192519)介紹到的docker container，會記得在my-pod的`/app`資料夾底下有API Server的原始程式碼與`Dockerfile`，我們可以用`ls`列出`/app`資料夾底下的所有檔案，如下圖所示

![kubectl-exec](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-exec.png?raw=true)


### 可以新增pod的labels

```
$ kubectl label pods <pod> <label-key>=<label-value>
```

可以先用`kubectl get pod --show-labels`查看目前`my-pod`有哪些labels，

```
$ kubectl get pods  --show-labels
NAME      READY     STATUS    RESTARTS   AGE       LABELS
my-pod    1/1       Running   0          14h       app=webserver
```

如果我們想要新增labels，可以輸入以下指令

```
$ kubectl label pods my-pod version=latest
pod "my-pod" labeled
```

再次用`kubectl get pod --show-labels`查看，可以發現，`LABELS`欄位多了一個version

```
$ kubectl get pod --show-labels
NAME      READY     STATUS    RESTARTS   AGE       LABELS
my-pod    1/1       Running   0          14h       app=webserver,version=latest
```

### 使用alpine查看cluster狀況

[alpine](https://alpinelinux.org/)提供非常輕量級的Docker Image，大小只有5MB上下。可以藉由在alpine下指令，從cluster中與其他pod互動，非常適合用來debug。以下面指令為例，在輸入下述指令後，我們就可以進到alpine的shell。

```
$ kubectl run -i --tty alpine --image=alpine --restart=Never -- sh
```

> 是否還記得我們今天提到，在Kubernetes Cluster中，會給每個Pod一個Cluster IP且只有在cluster裡才可以存取。而我們可以透過alpine在Kubernetes Cluster中訪問其他Pod。

可以用`kubectl describe`去查pod目前cluster IP，以`my-pod`為例，目前的cluster ip是`172.17.0.4`

```
$ kubectl describe pod my-pod
Name:         my-pod
Namespace:    default
Node:         minikube/192.168.99.104
....
Status:       Running
IP:           172.17.0.4
```

這時我們可以用`curl`指令去訪問Pod中container跑在port number 3000的API service。不過我們需要先在[alpine](https://alpinelinux.org/)中安裝`curl`套件。

```
$ apk add --no-cache curl
```
安裝完之後，我們可以輸入`curl http://172.17.0.4:3000`


![kubectl-apline-curl](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day05/kubectl-apline-curl.png?raw=true)

存取成功後，會看到`Hello World!`字串




# 總結
今天介紹kubectl常用指令以外，也認識了Kubernetes的第一個元件，[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)。之後的學習筆記中，會分享當有多個Pod同時在Kubernetes Cluster中時，我們會如何去管理、擴張以及確保每個Pod目前的運行狀態。





# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )




# 參考連結

 - [Kubernetes官網](https://kubernetes.io)
 - [Docker Hub](https://hub.docker.com/)
 - [Docker —— 從入門到實踐](https://philipzheng.gitbooks.io/docker_practice/content/)
