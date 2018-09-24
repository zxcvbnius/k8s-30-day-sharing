# 前言
在前幾天介紹完另外幾個常用的 Kubernetes 的元件後，想必讀者對 Kubernetes 有些基本了解。 今天將藉由這些元件，以及延續先前在 AWS 上架設的 Kubernetes Cluster，在 AWS 上架設 Stateful Wordpress Application，

> 若是對AWS 上如何架設 Kubernetes Cluster 還不太熟悉的讀者，不妨在開始今天實作之前，先來回顧一下幾個我們等等會使用的概念與物件：
>[Wordpress 是什麼 ? ](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13)
>[Stateless 與 Stateful 是什麼 ？](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07) 
>[在 AWS 上打造 Kubernetes Cluster (上)](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day15)
>[在 AWS 上打造 Kubernetes Cluster (下)](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day16)
>[如何動態產生 Kubernetes Cluster 所需的儲存資源](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day21)


今天學習筆記內容如下：

- 透過 [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/) 與 [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 動態產生 Mysql Data 儲存空間
- 透過 **aws-cli** 指令架設[一個可用的 nfs server](https://aws.amazon.com/tw/efs/)
- 用 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 物件存放我們所需的敏感資料
- 設定 [MySQL server](https://hub.docker.com/_/mysql/) YAML 檔
- 設定 [Wordpress](https://hub.docker.com/_/wordpress/) YAML 檔
- Demo

> 今天的程式碼都可以在 [demo-wordpress](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress) 上找到

# 透過 [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/) 與 [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 動態產生 Mysql Data 儲存空間

首先我們先需要定義一個 [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/)，以 [my-standard-storage.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/my-standard-storage.yaml) 為例，內容如下：

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zone: us-west-2a
reclaimPolicy: Delete
```

使用 `kubectl create` 產生一個新 Storage Class 物件，指令如下：

```
$ kubectl create -f ./my-standard-storage.yaml
storageclass "standard" created
```

用 `kubectl get` 指令查看，可以看到我們創建好的 `standard` 以外， AWS Cluster 本身也會幫我們創建其他 Storage Class，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-sc.png?raw=true)

若用 `kubectl describe` 則可看到該 Storage Class 的詳細資料，以 `standard` 為例，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-describe-sc.png?raw=true)

接著，我們還需要一個 [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 指定我們想要的儲存空間大小，以 [mysql-server-pvc.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/mysql-server-pvc.yaml) 為例，

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
```

一樣以 `kubectl create` 創建，

```
$ kubectl create -f ./mysql-server-pvc.yaml
persistentvolumeclaim "myclaim" created
```

接著用 `kubectl get` 查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-pvc.png?raw=true)

若是到 [AWS Console EC2 頁面](https://aws.amazon.com/tw/console/) 查看，也會發現多了一個 Volume，如下圖

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-ebs-info.png?raw=true)

如此，我們便將 Mysql Server 需要存放 data 的儲存區域架設好囉。

# 透過 **aws-cli** 指令架設[一個可用的 nfs server](https://aws.amazon.com/tw/efs/)

接著，在 Wordpress 上傳檔案或照片時，Wordpress Application 會將資料存放在 **/var/www/html/wp-content/uploads** 而不會存放在 MySQL server，因此我們還需要一個 nfs server 來幫我們存放這些資訊，指令如下：

```
$ aws efs create-file-system --creation-token WordpressFS
```

創建好之後，可以用 `aws-cli` 指令查看目前 `WordpressFS` 的狀態

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-cli-describe-efs.png?raw=true)

也可以到 [AWS Console EFS 頁面](https://aws.amazon.com/tw/console/)  查看，會發現多了一個我們剛剛創建好的 `WordpressFS`，且狀態為 `available`

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-efs-info.png?raw=true)
 

接著，我們需要知道目前 AWS 上 Kubernetes Cluster 的 `subnet ID`，可以透過`aws-cli` 指令找到

```
$ aws ec2 describe-instances
...
"SubnetId": "subnet-928491f4",
...
```

或者，我們也可以到 [AWS Console VPC 頁面](https://aws.amazon.com/tw/console/) 的 **Subnet** 選項中，找到我們需要的 subnet ID

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-subnet-info.png?raw=true)


接著，我們還需要知道 Cluster 上 EC2 的 Instance Security Group，如此我們新創建的 NFS 才能與 Node 互相溝通，一樣可以透過 `aws-cli` 或是從 [AWS Console EC2 頁面](https://aws.amazon.com/tw/console/)  的 **Security Group** 選項中，找到相對應的 `GroupId`，指令如下

```
$ aws ec2 describe-instances
...
"SecurityGroups": [
	{
		"GroupName": "nodes.k8sdemo.zxcvbnius.com",
		"GroupId": "sg-3f624a43"
	}
]
...
```

或是找到該頁面，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-security-group-info.png?raw=true)


接著，透過我們查詢到的 **NFS File System Id**，**GroupId**，與 **subnetId** 創建一個 mount target 供 Cluster 使用

```
$ aws efs create-mount-target \  
> --file-system-id fs-c7a7236e \
> --subnet-id subnet-928491f4 \
> --security-groups sg-3f624a43

{
    "FileSystemId": "fs-c7a7236e",
    ...
}
```

成功之後，回到 AWS Console EFS 的頁面中，可以看到底下多了一組 "Mount Target"，記下 **FileSystemId**，在之後的步驟我們將會需要使用。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-ebs-mount-target.png?raw=true)

# 用 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 物件存放我們所需的敏感資料

若是到 [Wordpress 官網](https://codex.wordpress.org/zh-cn:%E7%BC%96%E8%BE%91_wp-config.php) 查看，會發現 Wordpress 除了需要[我們上次設置](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13)的 **WORDPRESS_DB_PASSWORD** 以外，還需要 
 - **AUTH_KEY**
 - **SECURE_AUTH_KEY**
 - **LOGGED_IN_KEY**
 - **NONCE_KEY** 

這四種 key 增加使用上的安全

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/wordpress-security-page.png?raw=true)

因此，我們需要一個 `wordpress-secret` 去存放 Wordpress 需要的資料。以 [wordpress-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/wordpress-secret.yaml) 為例，

```
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secret
type: Opaque
data:
  db-password: cm9vdHBhc3M=
  auth_key: eEg+dn49UjlAMExEOHYsJmdNaD...
  secure_auth_key: eEg+dn49UjlAMExEOHYsJmd...
  logged_in_key: bkA0cTBSLkpSNl...
  nonce_key: PFZjSVB+RVBIYjJiQzZ4...
```


用 `kubectl create` 創建 `wordpress-secret`，

```
kubectl create -f ./wordpress-secret.yaml
secret "wordpress-secret" created
```

創建好後，可用 `kubectl get` 查看，可以發現 **DATA** 的部分代表我們存入了五個值

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-secret.png?raw=true)

# 設定 [MySQL server](https://hub.docker.com/_/mysql/) YAML 檔
在前置作業都完成後，接著我們可以開始來準備部署 MySQL server，如同[上次實做](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13)，這次我們一樣會使用 MySQL 官方提供的 Docker Image 來實做，[mysql-server-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/mysql-server-deployment.yaml) 內容如下，

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: mysql-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-server-deploy
  template:
    metadata:
      labels:
        app: mysql-server-deploy
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        args:
          - "--ignore-db-dir=lost+found"
        ports:
        - name: mysql-port
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: db-password
        volumeMounts:
         - mountPath: "/var/lib/mysql"
           name: mysql-server-storage
      volumes:
      - name: mysql-server-storage
        persistentVolumeClaim:
          claimName: myclaim
```

將我們剛剛建好的 `myclaim` 掛載到 `mysql-server` 底下，以 `kubectl create` 指令建立 **mysql-server**

```
$ kubectl create -f ./mysql-server-deployment.yaml
deployment "mysql-server" created
```

創建好後，也可以查看一下目前 Pod 的運行狀態，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-mysql-pod-1.png?raw=true)

接著，我們還需要一個 Service ，幫助 `mysql-server` 可以與其他物件溝通，以 [mysql-server-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/mysql-server-service.yaml) 為例

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
    app: mysql-server-deploy
  type: NodePort
```

以 `kubectl create` 創建一個新 [Service]() 物件， 

```
kubectl create -f ./mysql-server-service.yaml
service "mysql-server-service" created
```

`mysql-server-service` 會將收到的流量導給 labels 有 **app=mysql-server-deploy** 的 Pod。用 `kubectl get` 查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-mysql-server-service.png?raw=true)

# 設定 [Wordpress](https://hub.docker.com/_/wordpress/) YAML 檔

最後，在 MySQL server 架設好後，要來架設 Wordpress Application，[wordpress-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/wordpress-deployment.yaml) 設定檔如下：


```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: wordpress-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress-deploy
  template:
    metadata:
      labels:
        app: wordpress-deploy
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
          value: mysql-server-service
        - name: WORDPRESS_AUTH_KEY
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: auth_key
        - name: WORDPRESS_LOGGED_IN_KEY
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: logged_in_key
        - name: WORDPRESS_NONCE_KEY
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: nonce_key
        - name: WORDPRESS_SECURE_AUTH_SALT
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: secure_auth_key
        volumeMounts:
        - mountPath: /var/www/html/wp-content/uploads
          name: wordpress-uploads
      volumes:
      - name: wordpress-uploads
        nfs:
          server: fs-c7a7236e.efs.us-west-2.amazonaws.com
          path: /
```

透過 `kubectl create` 創建，

```
$ kubectl create -f ./wordpress-deployment.yaml
deployment "wordpress-app" created
```

確認都在運行後，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-pod-and-deploy.png?raw=true)

接著，我們還需要 `wordpress-service` 讓外部可以存取到 Wordpress Application，[wordpress-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/wordpress-service.yaml) 如下，

```
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
spec:
  ports:
  - port: 80
    targetPort: wordpress-port
    protocol: TCP
  selector:
    app: wordpress-deploy
  type: LoadBalancer
```


用 `kubectl create` 創建`wordpress-service`後

```
$ kubectl create -f ./wordpress-service.yaml
gitservice "wordpress-service" created
```

可以用 `kubectl get` 查看，會發現 **wordpress-service** 的 `EXTERNAL-IP` 為一組 LoadBalancer 代號

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/kubectl-get-svc.png?raw=true)

我們可以將這組 LoadBalander 代號在 Route53 綁定 `helloworld.k8sdemo.zxcvbnius.com` 如下圖，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/aws-route53-setting-age.png?raw=true)

接著，打開瀏覽器訪問 [helloworld.k8sdemo.zxcvbnius.com]()，便可以看到 Wordpress 的設定頁面囉

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/wordpress-setting-page.png?raw=true)

我們登入之後，可以在後台編輯頁面`上傳一張照片`或是`修改文字`，**[註一]**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/wordpress-add-photo.png?raw=true)

然後再**儲存更新**後，可以到首頁，便可以看到我們剛剛上傳的圖片囉

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/wordpress-demo.png?raw=true)

如此，即便 Kubernetes Cluster 中的 Container 消失，當下次有新的 container 掛載這些 Volumes 時，一樣能直接存取這些資料囉！

# 結論
希望藉由今天的實作中，讀者不但更能了解 Volume 的運用，也更能了解如何在 AWS 上操作 Kuernetes Cluster。在接下來的章節中，我們將會介紹 Kubernetes 更多更高級的元件運用，就請大家慢慢期待囉。

## [註一]
**Wordpress Docker Image** 本身有小小的 "bug"。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/wordpress-trouble-shooting.png?raw=true)

需要去更改 `/var/www/html/wp-content/uploads` 資料夾的權限才能將照片上傳上去。有兩種修改的方式，一種是在 [wordpress-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/demo-wordpress/wordpress-deployment.yaml) 加入修改權限的指令，另外一種則是進到 Pod 的 shell 去做修改，指令如下：

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day22/wordpress-modify-permission.png?raw=true)

# Q&A
最後，歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
- [AWS 白皮書](https://aws.amazon.com/tw/documentation/)
