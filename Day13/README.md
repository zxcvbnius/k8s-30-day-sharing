# 前言
在前幾天介紹完幾個常用的 [Kubernetes](https://kubernetes.io) 的元件後，想必讀者對 Kubernetes 有些基本了解。 今天我們將藉由這些元件，在 [minikube](https://github.com/kubernetes/minikube) 上架設 [Wordpress](https://tw.wordpress.org/)。

> 若是還不太熟悉的讀者，不妨在開始今天實作之前，先來回顧一下幾個我們等等會使用的物件：
>- [Stateless 與 Stateful 是什麼 ？](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07)
>- [ Kubernetes 最小運行單位 - Pod](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05)
>- [如何擴張我的 Pods?! - Deployments](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day08)
>- [外部服務與 Pods 的溝通管道 - Services](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09)
>- [將敏感資料存在 Kubernetes 中 - Secrets](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day12)

今天學習筆記內容如下：

- 介紹什麼是 Wordpress
- 從 Docker Hub 上下載 [Wordpress Image](https://hub.docker.com/_/wordpress/) 與 [MySQL Image](https://hub.docker.com/_/mysql/)
- 實作：如何在 minikube 上架設 Stateless Wordpress

> 小提醒：今天的程式碼都可以在 [demo-wordpress](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13/demo-wordpress) 上找到。


# 什麼是 Wordpress
Wordpress 是一個可以協助開發者快速架設好一個網站的軟體。Wordpress 是[開源的](https://tw.wordpress.org/)，以 PHP 和 MySQL 為主要架構。除了提供開發者許多外掛以及主題來打造個人 Blog、商業網站之外，Wordpress 本身還提供一個好用的後台，方便網站管理者修改、編輯文件。若是對於 Wordpress 還想暸解更多的讀者，不妨參考 [Wordpress 新手教學文件](https://startpress.cc/post/about-wordpress.html)。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/wordpress-homepage.png?raw=true)

# 從 Docker Hub 上下載 [Wordpress](https://hub.docker.com/_/wordpress/) 與 [MySQL](https://hub.docker.com/_/mysql/)
若有印象 [第四天學習筆記介紹到的 Docker Hub](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day04)，應該還記得 [Docker Hub](https://hub.docker.com) 上有者許多開源軟體提供的 Docker Image，而 [Wordpress 官方](https://hub.docker.com/_/wordpress/) 也有提供 Docker Image 讓開發者使用。

## Wordpress Docker Image
![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/docker-hub-wordpress.png?raw=true)

在 [Docker Hub for Wordpress 頁面中](https://hub.docker.com/r/library/wordpress/)，可以找到 Wordpress Image 中`可設定的環境變數(Environment Variables)` 的介紹。在稍後我們會透過 Secret 物件，將我們設定的環境變數傳給 Wordpress container。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/docker-hub-wordpress-how-to-use.png?raw=true)

## MySQL Docker Image
正如 [前言]() 提到，Wordpress 是 PHP 以及 MySQL 為主要架構的軟體。在 Wordpress Image 中提供了 PHP 的部分，MySQL server 則需要額外架設。

在這次範例中，我們將使用官方提供的 MySQL Docker Image，來存放 Wordpress 中的所有資料。
![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/docker-hub-mysql.png?raw=true)

在 [Docker Hub for MySQL 頁面](https://hub.docker.com/r/library/mysql/) 底下，也可以看到 `Environment Variables` 的介紹。

# 實作：如何在 minikube 上架設 Stateless Wordpress
首先，用 `kubectl get nodes` 確認目前 **minikube** 的運行狀態

```
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    1d        v1.8.0
```

接著，我們需要創建一個 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 物件，裡面存放著 Wordpress application 存取 MySQL server 的密碼，以 [wordpress-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13/demo-wordpress/wordpress-secret.yaml) 為例：

```
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secret
type: Opaque
data:
  # echo -n "rootpass" | base64
  db-password: cm9vdHBhc3M=
```

透過 `kubectl create` 創建 **wordpress-secret**，

```
$ kubectl create -f ./wordpress-secret.yaml
secret "wordpress-secret" created
```


接下來使用 [my-wordpress-deploy.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13/demo-wordpress/my-wordpress-deploy.yaml) 創建一個 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 物件去管理 Wordpress 服務， 內容如下：

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: wordpress-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress-deployment
  template:
    metadata:
      labels:
        app: wordpress-deployment
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
              name: wordpress-secret
              key: db-password
        - name: WORDPRESS_DB_HOST
          value: 127.0.0.1
      - name: mysql
        image: mysql:5.7
        ports:
        - name: mysql-port
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: db-password
```

在這個 `wordpress-deployment` 物件中，我們有兩個 container，分別是 Wordpress application 與 MySQL server。

- 在 Wordpress Docker Image 介紹中表示，開發者若無指定 `WORDPRESS_DB_USER`，則連結到 MySQL server 的 user 預設為 `root`

- 在 MySQL Docker Image 介紹中表示，可以透過 `MYSQL_ROOT_PASSWORD` 設定 `root` 帳號的密碼

- 因此，我們只需將 MySQL Docker Image 的 `MYSQL_ROOT_PASSWORD` 與 Wordpress Docker Image 中的 `WORDPRESS_DB_PASSWORD` 設置相同的密碼。Wordpress Container 就會以 **root 身份**連結到 MYSQL server container

- 我們將 Database 的密碼存在 `wordpress-secret` 中，並將密碼設置成 `環境變數`

- Pod 中的 container 都會透過 localhost + port number 互相溝通，所以 Wordpress container 會透過 **127.0.0.1:3306** 去訪問 MySQL server container

- 第一次 pull Image 需要較久的時間，可以用 `kubectl describe` 查看各個物件的狀態

接著，還需要創建一個 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 物件，讓本機端的瀏覽器也可以訪問到 Kubernetes Cluster 中的 `wordpress-deployment` 物件。[wordpress-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/demo-wordpress/wordpress-service.yaml) 設定如下：

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
    app: wordpress-deployment
  type: NodePort
```

透過 `kubectl create` 創建 **wordpress-service** ，

```
$ kubectl create -f ./wordpress-service.yaml
service "wordpress-service" created
```

最後，我們可以用 `kubectl get all` 來看一下目前所有物件的狀況，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/kubectl-get-all.png?raw=true)

當所有物件都 `READY` 時，透過 `minikube service` 指令找到 `wordpress-service` 的 url，

```
$ minikube service wordpress-service --url
http://192.168.99.100:30300
```

打開本機端的瀏覽器輸入 [http://192.168.99.100:30300](http://192.168.99.100:30300)

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/wordpress-welcomepage.png?raw=true)

便可以看到 Wordpress application 的設定頁面。選擇好自己所需的語言後，點選**繼續**，會進到使用者登入設定頁面，將 `網站名稱`、`使用者帳號`、`使用者密碼`，以及 `使用者email` 都輸入之後，往下點選 **Install Wordpress**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/wordpress-login-setting-page.png?raw=true)

進到登入頁面後，此時再輸入剛剛設定的帳號密碼，輸入完畢後點選 **Login**，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/wordpress-login-page.png?raw=true)

登入之後，便可以看到 Wordpress 的後台管理。點選左上角的 `My Blog` 可以進入到首頁。在首頁上，可以看到我們**剛剛設置好的使用者帳號與網站 title**，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day13/wordpress-my-blog-page.png?raw=true)

如此，我們就在 Kubernetes Cluster 中，利用 Wordpress Docker Image 與 MySQL Docker Image 架設好個人 Blog 網站了。

> 讀者不妨在 Wordpress 上試試看如何編輯或新增文章。更多有關 Wordpress 的教學，可參考 [該連結](https://codex.wordpress.org/zh-tw:WordPress_%E6%96%B0%E6%89%8B_-_%E5%A6%82%E4%BD%95%E9%96%8B%E5%A7%8B)

最後但也最重要的是，由於 MySQL 的資料都是存在 container 中，若是 Pod 不小心 crash 或是其他因素遭刪除，我們在 Wordpress 後台編輯的資料也會跟著遺失。因此在 [[Day 20] 如何保存我的資料 - Volumes]() 的學習筆記中，筆者也將會與大家詳細介紹如何透過 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) 保存 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 中的資訊。

# 總結
希望透過今天在 [minikube](https://github.com/kubernetes/minikube) 上架設 [Wordpress](https://tw.wordpress.org/) 的分享，能幫助讀者更熟悉 [Kubernetes](https://kubernetes.io) 的基本操作。而在往後的學習筆記中，筆者將介紹，如何透過第三方套件 [kops](https://github.com/kubernetes/kops) ，在AWS 上架設 Kubernetes Cluster，讓實作更貼近於一般的實際場景。


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )



# 參考連結
 - [Wordpress 官網](https://tw.wordpress.org/)
 - [Kubernetes 官網](https://kubernetes.io)
 - [Docker Hub](https://hub.docker.com)
