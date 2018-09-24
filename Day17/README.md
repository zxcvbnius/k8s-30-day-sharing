# 前言
在 [第九天的學習筆記中](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09) 我們學習到 **[Service](https://kubernetes.io/docs/concepts/services-networking/service/) 每次被建立時，Kubernetes Cluster 都會動態給予一組新的 Cluster IP**。然而，當我們的 Cluster 中有多個 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 同時運行，每個有各自的 [Services](https://kubernetes.io/docs/concepts/services-networking/service/) ，且彼此之間都用 [Services](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 互相溝通時，要如何確保 Service 在動態更新 Cluster IP 的情況下仍能找到彼此呢？幸好，這點 Kubernetes 也幫我們想到了。Kubernetes 內部提供一個 `kube-dns` 的插件，**讓我們可以不需要知道 Service 的 Cluster IP ，只透過 Service 的名稱，就能找到相對應 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)** 。

今天學習筆記如下：
- 介紹 [DNS](https://zh.wikipedia.org/zh-tw/域名系统) 是什麼
- 介紹 Kubernetes 內部插件 [kube-dns](https://github.com/kubernetes/dns)
- 實作：不同的 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 如何透過 [kube-dns](https://github.com/kubernetes/dns) 找到彼此

> 小提醒：今天的程式碼都可以在 [demo-dns](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns) 上找到唷

#  介紹 [DNS](https://zh.wikipedia.org/zh-tw/域名系统) 是什麼 ？
DNS，全名為 **Domain Name System**。DNS 會有一張表格，紀錄每個 [domain name](https://zh.wikipedia.org/zh-tw/%E5%9F%9F%E5%90%8D) 相對應的 IP 位址。如此，我們不再需要去紀錄該服務的 IP address，而是可以透過該服務的網域名稱連結到該服務。以 [www.google.com.tw](https://www.google.com.tw/) 為例，當我們在瀏覽器輸入 **www.google.com.tw** 時，[DNS](https://zh.wikipedia.org/zh-tw/域名系统) 會幫我們找到該網域相對應的 IP，並連結到該服務。

如果想知道某個特定的網域名稱相對應的 IP 位址，可以在終端機輸入 `host` 指令，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/host-command.png?raw=true) 

# 介紹 Kubernetes 內部插件 [kube-dns](https://github.com/kubernetes/dns)
而 Kubernetes 本身提供了 DNS 的套件，[kube-dns](https://github.com/kubernetes/dns)。[kube-dns](https://github.com/kubernetes/dns) 幫助在`同一個 Kubernetes Cluster 中的所有 Pods ，都能透過 Service 的名稱找到彼此`。 透過 `kubectl get` 指令，會發現 [kube-dns](https://github.com/kubernetes/dns) 也是一個在 Cluster 中運行的服務，一旦 Kubernetes Cluster 被建立後，便會自動運行。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/kubectl-get-kube-dns-pod.png?raw=true) 

而 `kube-dns` 的相關設定檔則放是放在 **master node 的 /etc/kubernetes/addons 資料夾底下**，以 [minikube](https://github.com/kubernetes/minikube) 為例，[minikube](https://github.com/kubernetes/minikube) 運行的 VM 本身就是 master node，所以我們先透過 `minikube ssh` 指令登入，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/kube-dns-configuration.png?raw=true) 

在 **/etc/kubernetes/addons** 資料夾底下，除了可以看到 kube-dns 的相關設定檔外，也可以看到 Kubernetes 其他套件的設定檔。

## [kube-dns](https://github.com/kubernetes/dns) 如何運作

如上圖我們可以看到 `kube-dns` 是運行在 Kubernetes Cluster 的一個 Pod，而這個 `kube-dns-86f6f55dd5-cht2h` Pod，也有一個相對應的 Service 物件。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/kubectl-get-kube-dns-service.png?raw=true) 

Kubernetes 在每一個 Pod 創建時，都會在該 Pod 的 `/etc/resolve.conf` 檔案中，自動加入 kube-dns service 的 domain name 與相對應的 IP 位址。因此 `其他 Pods 可以透過名稱為 kube-dns 的 Service 物件，找到正在運行的 kube-dns`，以下圖為例：

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/kube-dns-resolve.png?raw=true) 

我們在 Kubernetes Cluster 跑起 `alpine` Pod，並 ssh 進到 `alpine` 的 shell，用 `cat` 指令查看 **/etc/resolve.conf** 內容，

> 若是對 alpine 不太熟悉的讀者，不妨參考 [第五天分享的 kubectl 常用指令](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05)。

若是與上上圖比對，會發現 `/etc/resolve.conf` 中的 **nameserver** 指向的 IP 位置，就是 `kube-dns` 的 Service。


# 實作：不同的 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 如何透過 [kube-dns](https://github.com/kubernetes/dns) 找到彼此

以[先前介紹的 Stateless Wordpress](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13) 為例，先前我們是將 [Wordpress Image](https://hub.docker.com/_/wordpress/) 與 [MySql Image](https://hub.docker.com/_/mysql/) 放在同一個 Pod 中，現在我們將把他們拆開，放在不同的 Pod，並且透過相對應的 Service 名稱找到彼此。

今天我們分別會用到五個設定檔，[wordpress-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/wordpress-pod.yaml)，[mysql-server-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/mysql-server-pod.yaml)，[mysql-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/mysql-secret.yaml)，以及兩個 [service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/) 設定檔，

> 若是對如何在 Kubernetes Cluster 架設 Wordpress Application 還不太熟悉的讀者，不妨先看過 [第 13 天的學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13) 唷

# 建立 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) object
首先我們先建立 Secret，裡面存放著 mysql database 所需的密碼，[mysql-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/mysql-secret.yaml) 內容如下，


```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data: 
  # echo -n "rootpass" | base64
  db-root-password: cm9vdHBhc3M=
```

在 [mysql-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/mysql-secret.yaml) 中我們設置 `db-root-password` 為，`rootpass`。

以 `kubectl create` 創建該物件，指令如下：

```
$ kubectl create -f ./mysql-secret.yaml
secret "mysql-secret" created
```

創建好後，也可以用指令 `kubectl get secret` 查看我們創建的 `mysql-secret` 物件。

# 建立 Mysql Server

[mysql-server-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/mysql-server-pod.yaml) 內容如下，

```
apiVersion: v1
kind: Pod
metadata: 
  name: mysql-server
  labels:
    app: mysql-server
spec:
  containers:
  - name: mysql-server
    image: mysql:5.7
    ports: 
    - name: mysql-port
      containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: db-root-password
```

輸入 `kubectl create` 創建 **mysql** 物件，

```
$ kubectl create -f ./mysql-server-pod.yaml
pod "mysql-server" created
```

創建完後，可用 `kubectl get` 查看這個 Pod 是否正在運行，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/kubectl-get-mysql-pod.png?raw=true) 


接著，我們透過 [mysql-server-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/mysql-server-service.yaml) 這個設定檔創建一個 `mysql-server-service` 物件，內容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: mysql-server-service
spec:
  ports:
  - port: 3306
    protocol: TCP
  selector:
    app: mysql-server
  type: NodePort
```

一樣使用 `kubectl create` 創建，

```
$ kubectl create -f ./mysql-server-service.yaml
service "mysql-server-service" created
```

用 `kubectl get` 查看目前 `mysql-server-service` 的狀態

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day17/kubectl-get-mysql-server-service.png) 


# 建立 Wordpress App
在完成建置 Mysql Server 服務之後，我們將打造一個 Wordpress Application，透過 [wordpress-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/wordpress-pod.yaml) 與 [wordpress-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/wordpress-service.yaml)，內容分別如下：

[wordpress-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/wordpress-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  containers:
  - name: wordpress
    image: wordpress:4-php7.0
    ports:
    - name: wordpress-port
      containerPort: 80
    env:
    - name: WORDPRESS_DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: db-root-password
    - name: WORDPRESS_DB_HOST
      value: mysql-server-service
```


可以留意的是，`WORDPRESS_DB_HOST` 的部分我們是指定 `mysql-server-service` 的名稱，我們可以再創建一個 [wordpress-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/demo-dns/wordpress-service.yaml) ，讓 wordpress application 的服務可以透過本機端的瀏覽器瀏覽，內容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
spec:
  ports:
  - port: 3000
    nodePort: 30300
    protocol: TCP
    targetPort: wordpress-port
  selector:
    app: wordpress
  type: NodePort
```

透過 `kubectl create` 創建這兩個物件：

```
$ kubectl create -f wordpress-pod.yaml && \ 
> kubectl create -f wordpress-service.yaml
pod "wordpress" created
service "wordpress-service" created
```


完成指令後，我們可以用 `kubectl get all` 查看目前所有物件的狀態

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/kubectl-get-all.png?raw=true&version=2) 

確認所有物件都 `Ready` 狀態後，我們可以用 `minikube service` 來查看 `wordpress-service` 的 url 

```
$ minikube service wordpress-service --url
http://192.168.99.100:30300
```


接著在瀏覽器打開 [http://192.168.99.100:30300](http://192.168.99.100:30300)，便可以看到 wordpress 的歡迎頁面囉！

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/wordpress-welcome-page.png?raw=true) 

若有興趣嘗試更多的讀者，可以設定好帳號密碼後，進入 Wordpress 編輯，再進入 mysql server 看看，可以看到 database 有所改變唷。  

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day17/wordpress-page.png?raw=true)



# 結論
[kube-dns](https://github.com/kubernetes/dns) 使得 Pod 與 Pod 之間的溝通更方便。若有興趣更深入的讀者，不妨參考 [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) 這篇文章，相信會對 [kube-dns](https://github.com/kubernetes/dns) 的用法更加了解。


# 參考連結
 - [kube-dns](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns)
 - [Kubernetes 官網網站](https://kubernetes.io)
 - [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)


# 備註
感謝網友 @iming0319 勘誤 *kubectl get 創建* 應為 *kubectl create 創建*
