# [Day 9] 如何讓外部服務與pods的溝通管道 - Services






# 前言
若是還有印象前兩天介紹完 [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 以及 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 對 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 物件的操作，讀者應該可以發現[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 的生命週期是**非常動態**的。以`應用程式升級(rollout)` 為例，[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 會先創建新的 Pod 去取代現有的 Pod 物件，也因此，我們需要一個中間橋樑，來確保終端使用者(end user)或是其他應用服務，可以在該應用程式升級時，仍可以連到該應用程式中`可用的Pod (available)`。

今天學習筆記如下：

 - 什麼是 Service ？Service的用途有哪些 ？
 - 實作：透過 kubectl 操作 Service 物件
 - NodePort 在 Kubernetes 中的限制


> 小提醒：今天的程式碼都可以在 [demo-service](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09/demo-service) 上找到。






# 什麼是 Service ？Service的用途有哪些 ？
首先我們將介紹什麼是 [Service](https://kubernetes.io/docs/concepts/services-networking/service/)，如上述[前言]()所提到，我們需要在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 前面再接一層`橋樑`，確保每次存取應用程式服務時，都能連結到**正在運行的Pod**。若是還記得，我們前幾天常使用的指令`kubectl expose`，就會知道該指令可以幫我們創建一個新的 Service 物件，來讓Kubernetes Cluster中運行的 `Pod`與外部互相溝通。  

而一個新的 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 可以為 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 做到以下幾件事情：

 - 創建一個`ClusterIp`，讓Kubernetes Cluster中的其他服務，可以透過這個 **ClusterIp** 訪問到正在運行中的Pods
   在每次創建 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 物件時，Kubernetes 就會預設一組virtual IP 給 Service 物件。除非我們在 Service [yaml](https://zh.wikipedia.org/wiki/YAML) 指定想要的 virtual IP，否則 Kubernetes 每次都會隨機指定一組virtual IP。

 - 創建一個`NodePort`，讓**在 Kubernetes Cluster外但在同一個 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/)** 上的其他服務，可以透過這個 **NodePort** 訪問到 Kubernetes Cluster 內正在運行中的 Pods。

 - 如果我們的Kubernetes Cluster是架在第三方雲端服務(cloud provider)，例如 [Amazon](https://aws.amazon.com/tw/ec2/) 或 [Google Cloud Platform](https://cloud.google.com/compute/docs/instances/)，我們可以透過這些 cloud provider 提供的  [LoadBalancer](https://aws.amazon.com/tw/elasticloadbalancing/) ，幫我們分配流量到每個 Node 。我們可以指定`--type=LoadBalancer`，如此 cloud provider 就會自動幫我們創建好相對應的 Load Balancer 物件，在 [[Day 15] 介紹kops - 在AWS 上打造Kubernetes Cluster]() 將會介紹到如何透過指令創建 Load Balancer。





# 實作：透過 kubectl 操作 Service 物件

若是還記得[昨天透過 **kubectl expose** 的指令，為我們創建的 Service 物件](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day08)嗎？

```
$ kubectl expose deploy hello-deployment --type=NodePort --name=my-deployment-service
```

今天我們將透過 [my-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day09/demo-service/my-service.yaml) ，創建一個與`kubectl expose`相同行為的Service物件。

```
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  type: NodePort
  ports:
  - port: 3000
    nodePort: 30390
    protocol: TCP
    targetPort: 3000
  selector:
    app: my-deployment
```

 - **apiVersion**  
   `Service`使用的Kubernetes API是`v1`版本號

 - **metadata.name**  
   則可以指定該Service的名稱

 - **spec.type**  
   可以指定Service的型別，可以是`NodePort`或是`LoadBalancer`

 - **spec.ports.port**  
   可以指定，創建的Service的Cluster IP，是哪個port number去對應到`targetPort`

 - **spec.ports.nodePort**  
   可以指定`Node物件`是哪一個port numbrt，去對應到`targetPort`，若是在Service的設定檔中沒有指定的話，Kubernetes會隨機幫我們選一個port number

 - **spec.ports.targetPort**  
   targetPort是我們指定的 Pod 的 port number，由於我們會在Pod中運行一個port number 3000 的 web container，所以我們指定`hello-service`的特定port number都可以導到該web container。

 - **spec.ports.protocol**  
   目前 Service 支援`TCP`與`UDP`兩種protocl，預設為`TCP`

 - **spec.selector**  
   selector則會幫我們過濾，在範例中，我們創建的Service會將特定的port number收到的流量導向 `標籤為app=my-pod`的 Pods


> 若是細心的讀者，在閱讀前幾天介紹的 Replication Controller, Replication Set 或是 Deployment 的yaml檔時，可以發現 Kubernetes 在創建每個物件的設定檔都非常相似。


首先，一樣先確定`minikube`是否正常運作

```
$ kubectl get node
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    4d        v1.8.0
```

接著透過 [昨天我們使用的 my-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day09/demo-service/my-deployment.yaml) 創建一個`Deployment`物件，

```
$ kubectl create -f ./my-deployment.yaml
deployment "hello-deployment" created
```

確認`Pod`都在狀態都已經在`Running`之後，

```
$ kubectl get pods --show-labels
NAME                               READY     STATUS    RESTARTS   AGE       LABELS
hello-deployment-898fdcc4f-wdvn9   1/1       Running   0          29m       app=my-deployment,...
hello-deployment-898fdcc4f-wlr75   1/1       Running   0          29m       app=my-deployment,...
hello-deployment-898fdcc4f-zv274   1/1       Running   0          29m       app=my-deployment,...
```

我們就可以準備創建 Service 囉，指令如下


```
$ kubectl create -f ./my-service.yaml
service "hello-service" created
```

這時，我們可以透過`kubectl get services` 或是 `kubectl get svc`來查看目前service的狀態。

![kubectl-get-service](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day09/kubectl-get-service.png?raw=true)  

可以看到我們創建的`hello-service`物件，被賦予一組Cluster IP `10.111.116.249`，這時可以使用 [第五天學習筆記分享的alpine](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05) 來查看 Service在Cluster中是否正常運作，使用指令如下：

```
$ kubectl run -i --tty alpine --image=alpine --restart=Never -- sh
```

使用alpine查看cluster狀況

![kubectl-check-service](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day09/kubectl-check-service.png?raw=true)  

可以看到`curl 10.104.188.91:3000` 回傳 `Hello World!`字串，代表 `hello-service` 有正常運作。此外，我們也指定`NodePort`為 `30390`，可透過`minikube service`指令查看

```
$ minikube service hello-service --url
http://192.168.99.100:30390
```

可以透過本機端的瀏覽器打開[http://192.168.99.100:30390](http://192.168.99.100:30390)，也可以看到`hello-service`，我們從Kubernetes Cluster的外部服務訪問到 web app且回傳`Hello World`的字串。

![kubectl-nodeport-demo](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day09/kubectl-nodeport-demo.png?raw=true)  


#### Dynamic Cluster IP

如果我們，刪除目前的`hello-service`物件，又重新產生一個新的`hello-service`，會發現該物件的`Cluster IP`以及`minikube上的url`都會不同。這是因為這些virtual ip都是系統賦予的，除非我們在設定檔指定相對的`Cluster IP`，否則每次產生的Service物件的IP都會與先前不相同，以下述指令為例：

```
$ kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-service   NodePort    10.104.188.91   <none>        3000:30390/TCP   23m
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP          24m

$ kubectl delete svc/hello-service && kubectl create -f ./my-service.yaml
service "hello-service" deleted
service "hello-service" created


$ kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-service   NodePort    10.98.220.99   <none>        3000:30390/TCP   10s
kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP          56m
```

可以看到原本的CLUSTER-IP從`10.104.188.91`變為`10.98.220.99`





# NodePort 在 Kubernetes 中的限制
在Kubernetes預設的Service中，`Service可以指定的nodePort只有3000~32767`，如果我們需要修改可以指定的`nodePort range`，可以在 [Node](https://kubernetes.io/docs/concepts/architecture/nodes) 一開始創建中指定，以 `minikube` 為例，當下次啟動minikube時我們可以加上`--extra-config=apiserver.ServiceNodePortRange={PORT_RANGE}`

```
$ minikube stop && minikube start --extra-config=apiserver.ServiceNodePortRange=1-50000
....
```

啟動之後，我們可以將原本的 `hello-service` 編輯

![kubectl-set-node-port-range-demo](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day09/kubectl-set-node-port-range-demo.png?raw=true)  

這時再重新看一下 `hello-service` 的狀態，可以發現 NodePort 已經變為 `50000`了。

```
$ kubectl edit svc/hello-service
service "hello-service" edited

$ kubectl get svc hello-service
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-service   NodePort   10.98.220.99   <none>        3000:50000/TCP   14m

$ minikube service hello-service --url
http://192.168.99.100:50000
```






# 總結
今天的學習筆記中，分享了Kubernetes在實際場景中不可或缺的物件 - [Service](https://kubernetes.io/docs/concepts/services-networking/service/)。若是 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 是無法勝任在實際場景中的應用，讀者不妨想像一下，若是今天有兩個服務，一個是 web app 另一個是 database ，個別用 Deployment 物件管理並且透過 Service 物件提供端口服務，當 web app 要連到 database 時，我們要怎麼找到他的Cluster IP呢？在 [[Day 17] Pod之間是如何找到彼此呢 - Service Discovery]() 的學習筆記中，我們將會介紹到如何透過 [DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) 來幫助不同的應用服務發現彼此。





# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )





# 參考連結

 - [Kubernetes官網](https://kubernetes.io)
