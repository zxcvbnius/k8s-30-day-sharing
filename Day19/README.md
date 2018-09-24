# 前言
不知讀者是否還有印象 [第九天的學習筆記中](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09) 我們介紹到的 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 元件。Kubernetes 提供的 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 能幫助在 Cluster 中運行的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 能被外部服務存取。然而，每一個 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 物件都需將指定對外的 port number 與 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 上某個 port number 做 port mapping。這也代表，`當 Service 越來越多時，我們就需要管理更多的 port number，也會使得維運上更加複雜`。

而今天的學習筆記中，將介紹 Kubernetes 上的另一個設計元件，[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 。**透過 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 不但能使 Node 對外開放的 port 統一，結合 [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) 更能在 Kubernetes Cluster 中實現[負載平衡](https://zh.wikipedia.org/wiki/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1) 的功能**。

今天的學習筆記內容如下：

- 介紹什麼是 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- 進階：在 [minikube](https://github.com/kubernetes/minikube) 上架設 [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)


> 小提醒：今天的程式碼都可以在 [demo-ingress](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress) 找到

# 什麼是 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

若將 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 圖像化，可以看到當多個 Service 同時運行時，Node 都需要有相對應的 port number 去對應相每個  [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 的 port number，如下圖，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/describe-service.png?raw=true)


通常像 [AWS](https://aws.amazon.com/tw/) 或是 [GCP](https://cloud.google.com/?hl=zh-tw) 這樣的雲端服務，每台機器都會配置屬於它自己的防火牆(firewall)。這也代表，不論**新增、或是刪除 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 物件，我們都必須額外調整防火牆的設定，端口(port)的管理也相對複雜**。若是使用 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) ，我們**只需開放一個對外的 port number，[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 可以在設定檔中設置不同的路徑，決定要將使用者的請求傳送到哪個 Service 物件**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/describe-ingress.png?raw=true)

這樣的設計，除了讓運維者無需維護多個 port 或頻繁更改防火牆(firewall)外，`可以自設條件`的功能也使得請求的導向更加彈性。

藉由以下幾個範例，能讓我們更了解 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 的具體功能有哪些，

### Example 1：將不同路徑的請求對應到不同的 Service 物件
以 [ingress-example-1.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress/ingress-example-1.yaml) 為例，

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-1
spec:
  rules:
  - http:
      paths:
      - path: /test
        backend:
          serviceName: test
          servicePort: 80
```
 
由上述 [YAML](https://zh.wikipedia.org/zh-tw/YAML) 設定檔我們可以知道，
- **目前 Ingress 支援的 API 版本** 是 `extensions/v1beta1`
- 該設定檔中設定了一個規則：Node 收到流量之後，判斷流量路徑，  
  若是請求路徑為 `/test` 則該流量將**導到名稱為 test 的 Service 物件**

創建新 Ingress 物件的方式，與其他元件一樣，使用 `kubectl create` 指令：

```
$ kubectl create -f ./ingress-example-1.yaml
ingress "example-1" created
```

創建好用，用 `kubectl get` 查看

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-get-example-ingress-1.png?raw=true)


### Example 2：將不同 domain name 的請求對應到不同的 Service 物件
以 [ingress-example-2.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress/ingress-example-2.yaml) 為例，

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-2
spec:
  rules:
  - host: helloworld-v1.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: hellworld-v1
          servicePort: 80
  - host: helloworld-v2.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: helloworld-v2
          servicePort: 80
```

若是有`多個 Domain Name 同時指向一台 Node` 時，我們也可以透過這樣路徑的設置，將不同的 Domain Name 對應到不同的 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 物件。

用 `kubectl create` 創建 `example-2`

```
$ kubectl create -f ./ingress-example-2.yaml
ingress "example-2" created
```

在與上圖比對可以發現，不同於 `example-1` 收到請求後若是路徑相符，會直接將請求導給 `test` 物件，`example-2` 會**判斷該請求是送到哪個 Domain Name，再將該請求導到對應的 Service 物件**。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-get-example-ingress-2.png?raw=true)

### Example 3：支援終止 SSL (SSL termination)
[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 也可以做到`本地終止 SSL (SSL termination)`。

首先，先透過 [ingress-ssl-sceret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress/ingress-ssl-sceret.yaml) ，將 [SSL 憑證](https://www.youtube.com/watch?time_continue=43&v=UNImBt5tTlg)存入 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 物件，


```
apiVersion: v1
data:
  tls.crt: base64_encoded_cert
  tls.key: base64_encoded_key
kind: Secret
metadata:
  name: ssh-secret
  namespace: default
type: Opaque
```

創建好後，可以在 Ingress 物件中，透過 `spec.tls` 將該憑證掛載載 Ingress 底下，以 [ingress-example-3.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress/ingress-example-3.yaml) 為例

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-3
spec:
  tls:
  - secretName: ssh-secret
  backend:
    serviceName: apiservice
    servicePort: 80
```


在對 Ingress，有些基礎了解後，以下將介紹 Ingress Controller。並**透過在 `minikube` 上架設 Nginx Ingress Controller 的實作**，來幫助讀者更暸解這兩者的應用。

# 在 [minikube](https://github.com/kubernetes/minikube) 上架設 [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)
**[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 本身並沒有提供[負載平衡](https://zh.wikipedia.org/wiki/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1)的功能，還需要透過 [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) 來實現**。Ingress Controller 目前主要支援兩種型別 [GCE](https://git.k8s.io/ingress-gce/README.md) 與 [Nginx](https://git.k8s.io/ingress-nginx/README.md)，而今天我們將透過 [Nginx Ingress Controller](https://git.k8s.io/ingress-nginx/README.md) 在 Kubernetes Cluster 內部架設 load balancer。

> 負載平衡(Load Balancing)：**以往我們可以透過外部的資源，像是 [AWS ELB](https://aws.amazon.com/tw/elasticloadbalancing/) 來實現，將流量分配給不同的機器。**而 Kubernetes 現在也提供了一個內部工具 [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) ，讓我們自己可以在 Kubernetes Cluster 中實現 Load Balancing 而無需透過外部資源。

## 創建 Hello World Application
我們用 [helloworld-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress/helloworld-pod.yaml) 來創建一個 `helloworld-pod` 的物件內容如下，

```
apiVersion: v1
kind: Pod
metadata:
  name: helloworld-pod
  labels:
    app: helloworld-pod
    tier: backend
spec:
  containers:
  - name: api-server
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000
```


這時再用 `kubectl create` 指令，

```
$ kubectl create -f ./helloworld-pod.yaml
pod "helloworld-pod" created
```

接著透過 [helloworld-service](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-ingress/helloword-service.yaml) 創建 `helloworld-pod` 相對應的 Service 物件，


```
apiVersion: v1
kind: Service
metadata:
  name: helloworld-service
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: helloworld-pod
```


一樣用 `kubectl create` 產生一個新 Service 物件，

```
$ kubectl create -f ./helloworld-service.yaml
service "helloworld-service" created
```


用 `kubectl get` 查看目前的狀態

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-get-hello-world-service.png?raw=true)

接著我們可以到 [Nginx Ingress Controller 的 Github 上](https://github.com/kubernetes/ingress-nginx)，進到 [deploy](https://github.com/kubernetes/ingress-nginx/tree/master/deploy) 這個資料夾。只需要按照該資料夾的 README.md 上的步驟，即可建置完成。

### Create ingress-nginx namespace
首先，創建一個名為 `ingress-nginx` 的 namespace。指令如下，

```
$ curl https://raw.githubusercontent.com/kubernetes\
>/ingress-nginx/master/deploy/namespace.yaml \
>| kubectl apply -f -
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-create-namespace.png?raw=true)

> 若是對 Namespace 還不熟悉的讀者也無需擔心，在 [[Day 28] 如何在 k8s 管理不同的專案 - Namespaces]() 會在與讀者詳細介紹 Namespace 是什麼。

### Create default backend
Kubernetes 官方提供我們一個 `default-backend`。當 Nginx Ingress Controller 找不到相對應的路徑時，一律回傳 `default backend - 404`

```
$ curl https://raw.githubusercontent.com/kubernetes\
>/ingress-nginx/master/deploy/default-backend.yaml \
>| kubectl apply -f -
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-create-default-backend.png?raw=true)

### Create ConfigMaps
接著在創建幾個所需的 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)，

```
$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml | kubectl apply -f -

$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml | kubectl apply -f -

$ curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml | kubectl apply -f -
```


### Create Nginx Ingress Controller
最後，創建一個 Nginx Ingress Controller

```
$ curl https://raw.githubusercontent.com/kubernetes\
>/ingress-nginx/master/deploy/without-rbac.yaml \
>| kubectl apply -f -
```

可以看到在官網提供的範例中，使用 Deployment 來確保 Ingress Controller 的運行狀態。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-create-nginx-ingress-controller.png?raw=true)

全部都設置好後，用 `kubectl get` 查看 我們剛剛所創建的物件

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/kubectl-get-all-in-ingress-nginx-namespace.png?raw=true)


## Create Helloworld Ingress
接著，我們還需一個 Ingress，[helloworld-ingress.yaml]() 內容如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: helloworld-ingress
  namespace: default
spec:
  rules:
  - host: helloworld.example.com
    http:
      paths:
      - backend:
          serviceName: helloworld-service
          servicePort: 3000
```

把 `helloworld.example.com` 收到的請求，都轉到 `helloworld-service` 上。

### Enable on minikube
最後，若是在 `minikube` 上實作的讀者，不忘開啟 minikube 上 ingress 的功能

```
$ minikube addons enable ingress
ingress was successfully enabled
```

並且用 `minikube ip` 取得目前 minikube 的 IP address

```
$ minikube ip
192.168.99.100
```

### 設置本機端的 Host
最後，我們須將本機端**發向 `helloworld.example.com`** 的請求，都導到 `minikube` 上，指令如下：

```
$ sudo nano /etc/host
```

使用 **MacOS** 或 **Linux** 的讀者，都能在 `/etc/host` 這個檔案中找到，特定的 domain name 相對應的 ip adress。以下圖為例，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/localhost-domain-ip-table.png?raw=true)

最後加上我們剛剛查訊到的 minikube ip 與 `helloworld.example.com` ，如此一來，當我們在本機上搜尋 `helloworld.example.com` 時，系統就會根據 `/etc/host` 的內容，將請求導到 minikube ip address。

### Demo
在一切都設定完之後，可以打開瀏覽器輸入 [http://helloworld.example.com/](http://helloworld.example.com/)，可以看到網頁回傳 `Hello World!` 的字串，代表 `helloworld-ingress` 將流量成功導到 `helloworld-pod`，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-hello-world.png?raw=true)

如果讀者還沒忘的話，在架設 [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx) 的時候，我們也創建了一個 `default-backend` 的物件。用來處理，當 Ingress Controller 找不到相對應的 Ingress Rule 時，會預設回傳 `default backend - 404`。如下圖：

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day19/demo-default-backend.png?raw=true)

在 [Nginx Ingress Controller 的 Github](https://github.com/kubernetes/ingress-nginx) 中，也有詳細介紹如何將 Ingress Controller 部署在 [Azure](https://azure.microsoft.com/zh-tw/)，[AWS](https://aws.amazon.com/tw)，[GCP](https://cloud.google.com/?hl=zh-tw) 上，若有興趣的讀者不妨看一看囉。

# 總結
目前 [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) 仍存在一些 [issues](https://github.com/kubernetes/ingress-nginx/issues)，若是想使用 Ingress Controller 取代外部資源的 Load balancing 的讀者，不妨先仔細看過 [README.md](https://github.com/kubernetes/ingress-nginx) 減少踩雷的痛苦唷。


# Q&A
最後，歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
