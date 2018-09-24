# 前言
不知大家在開發的過程中是否遇過，將部署環境的代碼與程式碼一併交付的經驗。筆者在最一開始自身開發產品時，也犯了這樣的錯誤，將`不同環境(development, staging, production)`的部署代碼一併交付在程式碼中，而沒有發覺自己將應用服務暴露在危險之中，**一旦其他人也存取代碼，便能知道各個環境的敏感資料**。在 [12 factor](https://12factor.net/config) 中針對這樣的情況提出一個解決的方法：`將配置環境儲存於環境變數中`，降低 Configuration 與程式碼的耦合，減少環境代碼被交付在程式碼的情況。然而，將環境代碼與程式碼分開之後，我們也會面臨到另外一個問題：環境代碼(Configuration)可能通常都散落在各地，必須`有個的地方統一管理這些環境代碼`。更糟的是，每個環境代碼可能都因應不同的程式語言，有著不同的格式，也使得統一管理更加困難。

而 [Kubernetes](https://kubernetes.io) 本身提供了一個 API - [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) ，不但提供一個 Configuration 可以統一存放的地方，也提供一個方法讓開發者可以`動態且代碼化`的方式為每個應用服務配置其相對應的 Configuration。

今天的學習筆記如下：
- 定義什麼是 Configuration
- 比較 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 與 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 的差別
- 介紹 ConfigMap
- 實作：如何透過 ConfigMap 配置 [Nginx](https://nginx.org/en/) 


> 小提醒：今天的程式碼都可以在 [demo-configmap](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap) 中找到唷

# 定義什麼是 Configuration
在今天的學習筆記中，提到的 Configuration 泛指`程式存取外部資源或是部署所需的資料`，像是**資料庫的所在 IP、管理者的帳號密碼，或是 Nginx 的設定檔**等等。若是其他人不小心存取資源，導致資料庫被刪除，或被公開，都可能導致專案陷入危險之中，或是損害該應用服務的使用者的權益。

# 比較 ConfigMap 與 Secret 的差別
在先前我們介紹過 [[Day 12] 敏感的資料怎麼存在k8s?! - Secrets](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day12) ，也許讀者會好奇這兩者的差異是什麼。事實上 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 與 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 想解決的面向不太相同，我們可以將機密的資料存在 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 中，且 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 會將這些值經過 [Base64 加密](https://en.wikipedia.org/wiki/Base64)，機密的資料像是 API 或是 database 的密碼；而將非機密但屬於部署面的資料放在 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)，好比資料庫的 port number 或是 Redis 的 config file。 

# 介紹 ConfigMap
[ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 有以下特點：

- **一個 ConfigMap 物件可以存入整個 configuration**   
  像是 webserver config file, [Nginx](https://nginx.org/en/) config file 
- **無需修改 container 程式碼，可以替換不同環境的 Config**  
  開發過程中，常因應不同的環境需配置不同的 configuration，像是 `staging` 與   `production` 存取的資料庫位址不一致等等。無需修改程式碼的特點，可以幫助我們更快部署到各個不同的環境中。
- **統一存放所有的 configuration**  
  透過 `kubectl get` 指令快速查看目前系統所有的 ConfigMap。
  
## 建置 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 
建置一個 ConfigMap 物件也有以下兩種方式，

### 匯入整個 config 檔
ConfigMap 提供 `--from-file` 參數，讓我們可以存入整個檔案，以 [Redis](https://redis.io/) config file 為例，以下是 [my-redis.conf](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap/my-redis.conf) 的內容，

```
bind 127.0.0.1
port 6379
maxclients 10000
maxmemory 50mb
maxmemory-policy volatile-lru
syslog-enabled yes
dir /var/lib/redis
dbfilename redis.dump.rdb
databases 1
appendfsync everysec
save 600 10
```

透過 `kubectl create` 創建，

```
$ kubectl create configmap redis-config --from-file=my-redis.conf
configmap "redis-config" created
```

創建好後，可以用 `kubectl get` 指令查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/kubectl-get-configmap.png?raw=true)

可以看到我們剛剛創建好的 `redis-config` ，如果用 `kubectl describe` 則可以看到該物件更詳細的資料，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/kubectl-describe-redis-conf.png?raw=true)

可以看到我們剛剛創建的 `redis-config` 中，**my-redis.conf** 中的資料。


### 從指令設定 Config

而我們也可以在 `kubectl create` 指令中，直接設定 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 的值，以述下指令為例，

```
$ kubectl create configmap mysql-host --from-literal=ip=127.0.0.1

configmap "mysql-host" created
```

若從 `kubectl describe` 指令中查看，也可以看到我們新創建好的 `mysql-host` 中的內容，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/kubectl-describe-mysql-host.png?raw=true)

最後，如果我們想移除掉某個特定的 ConfigMap 物件，可以使用 `kubectl delete` 來刪除，指令如下：

```
$ kubectl delete configmap mysql-host
configmap "mysql-host" deleted
```

如此，再用 `kubectl get` 指令查看，就會發現只剩下 `redis-config` 囉

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/kubectl-get-configmap-before-deleting.png?raw=true)

# 實作：如何透過 ConfigMap 配置 [Nginx](https://nginx.org/en/) 
在今天的實作中，將透過 ConfigMap 配置 Nginx Service，讓 Nginx 在收到 request 之後，都能將流量導到 [我們第三天打造的 Nodejs APP](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03)。首先，我們要先知道 [Nginx](https://nginx.org/en/) 的用途，

## 什麼是 Nginx
[Nginx](https://nginx.org/en/) 不只是一套輕量的 HTTP 伺服器，同時也是一個[反向代理](https://zh.wikipedia.org/zh-tw/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86)的伺服器。收到用戶的請求後，可以將流量導給後端的 service，再將後端處理好的資源回傳給前端使用者。若是對於 [Nginx](https://nginx.org/en/) 不妨參考[官方提供的 Guide ](http://nginx.org/en/docs/beginners_guide.html)，相信會對 Nginx 的了解更加豐富。

## Nginx 配置檔案
今天我們會使用到兩個檔案，一個是 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap/my-pod.yaml) 另外一個是 [my-nginx.conf](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap/my-nginx.conf) ，[my-nginx.conf](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap/my-nginx.conf) 內容如下：

```
server {
    listen            80;
    server_name       localhost;

    location / {
        proxy_bind 127.0.0.1;
        proxy_pass http://127.0.0.1:3000;
    }

    error_page 500 502 503 504    /50x.html;
    location = /50x.html {
        root    /usr/share/nginx/html;
    }
}
```


我們將透過 [my-nginx.conf](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap/my-nginx.conf) 創建一個 ConfigMap 供 Nginx container 使用，指令如下：

```
$ kubectl create configmap nginx-conf --from-file=./my-nginx.conf
configmap "nginx-conf" created
```


## Nginx 與 API server 的配置

創建好後，我們可以看 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-configmap/my-nginx.conf) 的內容，

```
apiVersion: v1
kind: Pod
metadata:
  name: apiserver
  labels:
    app: webserver
    tier: backend
spec:
  containers:
  - name: nodejs-app
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000
  - name: nginx
    image: nginx:1.13
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-conf-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-conf-volume
    configMap:
      name: nginx-conf
      items:
      - key: my-nginx.conf
        path: my-nginx.conf
```


輸入指令 `kubectl create` 指令，

```
$ kubectl create -f ./my-pod.yaml
pod "apiserver" created
```

創建完後，用 `kubectl get` 查看目前 `apiserver` 的運行狀態是否 **Ready**，


![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/kubectl-get-pods.png?raw=true)

確定狀態後，使用 `kubectl expose` ，指定將 port 80 對應到 minikube 上的某一個 port，

```
$ kubectl expose pod apiserver --port=80 --type=NodePort
service "apiserver" exposed
```

再用 `minikube service` 指令讓本機端可以直接從瀏覽器送請求到 `apiserver` 上，

```
$ minikube service apiserver --url
http://192.168.99.100:31529
```

在瀏覽器打開 [http://192.168.99.100:31529](http://192.168.99.100:31529) 會發現，出現 `Hello World!` 的字串，代表 ConfigMap 成功的掛載在 Nginx Pod 上囉！

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day18/demo-hello-world.png?raw=true)


# 總結
在今天學習筆記最後，我們用`Volumes 掛載(mount)` 的方式將 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 放在 Pod 上。若是不熟悉 Volumes 的讀者也無須擔心，在之後 [[Day 20] 如何保存我的資料 - Volumes]() 學習筆記中，將會更進一步介紹 Kubernetes 的另外一個元件 -  [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)。

# 參考連結
- [Redis 官網](https://redis.io/)
- [12factor](https://12factor.net/config)
- [Nodejs 官網](https://nodejs.org/en/)
- [Configuration management with Containers](http://blog.kubernetes.io/2016/04/configuration-management-with-containers.html)
