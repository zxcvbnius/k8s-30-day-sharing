# 前言
在 [前一天](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day15) 我們知道了：
- 如何在 MacOS 與 Linux 安裝 [kops](https://github.com/kubernetes/kops) 與 [awscli](https://aws.amazon.com/tw/cli/) 套件
- 如何在 [AWS](https://aws.amazon.com) 上設置好我們需要的設定

今天的學習筆記將分享：
- 如何透過 kops 指令建造 Kubernetes Cluster
- 部署 [Docker Image](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day04) 到 AWS 上
- 在 AWS 上實現 [Load balancer](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09) 功能
- 如何透過 kops 刪除 Kubernetes Cluster

> 小提醒：今天的程式碼都可以在 [demo-k8s-on-aws](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/demo-k8s-on-aws) 上找到唷

# 如何透過 kops 指令建造 Kubernetes Cluster
設置好一切後，我們須先產生一組 `ssh key`，日後我們想登入 Kubernetes Cluster 中時，都需要透過這把 `ssh key` 才能登入。

## 產生 SSH-key
透過以下 [ssh-keygen 指令](https://en.wikipedia.org/wiki/Ssh-keygen) 可以在本機端產生一組 **ssh key**，且我們將這組 Key 放置在 `$HOME/.ssh` 底下，

```
$ ssh-keygen -f .ssh/id_rsa
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/ssh-keygen-create-ssh-key.png?raw=true)

產生完之後，我們可以用 `cat` 指令查看 **publick key** 與 **private key**，

```
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAA....

$ cat ~/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
....
-----END RSA PRIVATE KEY-----
```

接著，透過 `kops` 指令在 AWS 上建置 Kubernetes Cluster, 指令內容如下，

```
$ kops create cluster \
> --name=k8sdemo.zxcvbnius.com \
> --state=s3://k8s-demo-qwer \
> --zones=us-west-2a \
> --master-size=t2.micro \
> --node-size=t2.micro \
> --node-count=2 \
> --dns-zone=k8sdemo.zxcvbnius.com
```

- **name**  
  指定該 Kubernetes cluster 的名稱
  
- **state**  
  Kubernetes 的資訊將存放在 S3 `k8s-demo-qwer` 這個 bucket 中
  
- **zones**  
  是指定這些 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 要部署在哪些區域(Region)，在[這裡](https://docs.aws.amazon.com/general/latest/gr/rande.html)，可以找到 AWS 上所有`可以指定的區域`。以筆者而言，指定在 `US West(Oregon)`，相對應的區域是 **us-west-2**
  
- **master-size**  
  指定 master-node 的機器型別，在 [這裡](https://aws.amazon.com/ec2/instance-types/)，你可以找到 AWS 上所有`可指定的機器型別`，以及每個不同類型機器的規格介紹。
  
- **node-size**  
  如同，`master-size`，這裡是指定一般 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 的機器型別。
  
- **node-count**
  指定 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 的數量。

輸入指令之後，我們可以預覽在 AWS 上將有的變化，更多的 log 可參考 [該連結](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/demo-k8s-on-aws/kops-create-k8s-cluster.log) 

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/kops-preview-aws-changes.png?raw=true)

如果發現有想要修改的地方，可用 `kops edit` 。確認都無誤時，輸入 `Finally configure your cluster with` 字串後面指令，以及指定 `--state` ，

```
$ kops update cluster k8sdemo.zxcvbnius.com --yes --state=s3://k8s-demo-qwer
```

完成之後，我們可以用 `cat` 指令，來查看 `$HOME/.kube/config` 底下的設定檔。

```
$ cat ~/.kube/config
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/k8s-api-server-info-2.png?raw=true&version=2)
  
可以看到我們的 apiServer 是，`https://api.k8sdemo.zxcvbnius.com/` 。開啟瀏覽器，用下面的帳號密碼登入，便可看到 Kubernetes 所有可存取的 API。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/api-server-webpage.png?raw=true)

若是我們在看 [AWS console](https://console.aws.amazon.com) 上的變化，會發現 AWS 都幫我們設置好了。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-changes-ec2.png?raw=true)

在指定的 Region，分別產生 **master** 與 **node**，也在 Route53 都幫我們設定好相對應的路徑，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-changes-route53.png?raw=true)


以上述的 apiServer 為例，可以看到 **https://api.k8sdemo.zxcvbnius.com/ 指向 master node 的 IP address**。

最後，可以看到 S3 `k8s-demo-qwer` bucket 中多了一個 `k8sdemo.zxcvbnius.com` 欄位。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-changes-s3.png?raw=true)

點進去後，可以看到該欄位中存放著 Kubernetes 的相關設定資訊

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-changes-s3-2.png?raw=true)
  
最後，我們一樣能用 `kubectl` 指令查看目前 Kubernetes 上物件的狀態，以 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 為例，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/kubectl-get-nodes.png?raw=true)

#  部署 [Docker Image](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day04) 到 AWS 上
部署方式一樣是透過 `kubectl 指令`，以 [my-deployment.yaml] 為例：
 
```
apiVersion: apps/v1beta2
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
        app: my-deployment
    spec:
      containers:
      - name: webapp
        image: zxcvbnius/docker-demo:latest
        ports:
        - name: webapp-port
          containerPort: 3000
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/kubectl-create-my-deployment.png?raw=true&version=2)

完成之後，查看目前 `my-deployment` 的狀態，

```
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-deployment   3         3         3            3           5m
```

我們透過 `kubectl expose` 創建一個 Service 物件，讓外部可以透過 Master Node 存取到 `hello-deployment`，指令如下：

```
$ kubectl expose deploy hello-deployment\ 
> --type=NodePort --name=hello-deployment-service
service "hello-deployment-service" exposed
```

用 `kubectl get` 查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/kube-expose-service.png?raw=true)

如此，我們便能透過 Master Node 的 port number 31769 存取 `hello-deployment`。

需要注意的是，AWS 對外的沒有開放 port number 31769，因此我們需要手動更改防火牆，讓外部的使用者也可以連結到這個 port 。先到 [AWS EC2 console 的頁面](https://us-west-2.console.aws.amazon.com/ec2) 點選 **Security Groups**，再點擊 **master server**，底下可以看到 **inbound** 的選項，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-preview-security-group.png?raw=true)

點選 **Edit**，新增一組 TCP Rule, 開放 port number 31769 給外部連線。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-edit-security-group.png?raw=true)
  
設定完成後，我們可以在原先 [AWS EC2 console 的頁面](https://us-west-2.console.aws.amazon.com/ec2) 找到 master node 的網址。打開瀏覽器查看，可以看 `hello-deployment` 到回傳的 `Hello World!` 字串。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/master-node-hello-world.png?raw=true)

# 在 AWS 上實現 [Load balancer](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09) 功能

接著我們將透過 2 個會回傳不同字串的 Pod，來展示 AWS Load balancer 的功能。[hello-pod-1](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/demo-k8s-on-aws/hello-pod-1.yaml) 會回傳 **Hello World!** 字串，內容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod-1
  labels:
    app: webserver
    tier: backend
spec:
  containers:
  - name: pod-demo
    image: zxcvbnius/docker-demo
    ports:
    - name: hello-pod-port
      containerPort: 3000
```

而 [hello-pod-2](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/demo-k8s-on-aws/hello-pod-2.yaml) 則會回傳 **Hello World! 2** 字串，內容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod-2
  labels:
    app: webserver
    tier: backend
spec:
  containers:
  - name: pod-demo
    image: zxcvbnius/docker-demo:v2.0.0
    ports:
    - name: hello-pod-port
      containerPort: 3000
```

使用 `kubectl create` 來創建這兩個 Pod 物件，

```
$ kubectl create -f ./hello-pod-1.yaml
pod "hello-pod-1" created

$ kubectl create -f ./hello-pod-2.yaml
pod "hello-pod-2" created
```

接著我們創建一個 Service 物件，會將流量導給 Label 有 `app=webserver` 的 Pod，[hello-pod-service.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/demo-k8s-on-aws/hello-pod-service.yaml) 內容如下，

```
apiVersion: v1
kind: Service
metadata:
  name: hello-pod-service
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: hello-pod-port
  selector:
    app: webserver
  type: LoadBalancer
```

一樣用 `kubectl create` 創建 `hello-pod-service` 物件，

```
$ kubectl create -f ./hello-pod-service.yaml
service "hello-pod-service" created
```

接著，我們可以在 [AWS EC2 console 的頁面](https://us-west-2.console.aws.amazon.com/ec2) 點選 **EC2 Dashboard** 可以發現多了一個 **Load Balancers**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-lb-1.png?raw=true)

點選 **Load Balancers** 之後，可以看到 **Port Configuration**，指定將收到的 port 3000 的流量導給每個對應的 Node 的 port number 30156

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-lb-2.png?raw=true)

點擊 **Instance** 可以看 Load balancer 到對應到的機器有哪些，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-lb-3.png?raw=true)

如上述所提，Load Balancer 會把流量導給每個 Node 的 port number 30156。**Load Balancer 不需要知道哪個 Node 上有哪些 Pod，Kubernetes 都會幫我們處理**。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-lb-4.png?raw=true)

接著到 [AWS Routes53 的設定頁面](https://console.aws.amazon.com/route53)，新增一個 DNS 給這個 Load Balancer。在這裡，筆者設定 `helloword.k8sdemo.zxcvbnius.com` 為 Load Balancer 的 domain name

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-route53-setting-1.png?raw=true)

完成之後，我們可以看到設定好的值

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-route53-setting-2.png?raw=true)

接著打開瀏覽器，輸入 [http://helloword.k8sdemo.zxcvbnius.com:3000/](http://helloword.k8sdemo.zxcvbnius.com:3000/) 

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-route53-setting-3.png?raw=true) 

便可以看到 `hello-pod` 回傳回來的 `Hello World!` 的字串囉！如果多試幾次，也可以看到 `Hello World! v2` 的字串，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day16/aws-route53-setting-4.png?raw=true) 

# 透過 kops 刪除 Kubernetes Cluster
以上是我們今天的實作，如果結束之後，想把 Kubernetest Cluster 關掉不想再被 AWS 收取費用，可以用以下指令：
```
$ kops delete cluster \ 
> --name=k8sdemo.zxcvbnius.com \ 
> --state=s3://k8s-demo-qwer \ 
```

最後確認無誤後，再輸入 `--yes`。如此 [kops](https://github.com/kubernetes/kops) 就會幫我們清掉剛剛在 AWS 創建的那些資源了，包含 master server, node, 以及 Load balancer 等。 

# 結論
筆者在學習的過程中，最害怕的是不知道怎麼將學到的技術運用在實際場景上。而 [kops](https://github.com/kubernetes/kops) 大大減少了開發者一開始在雲端服務上架設 kubernetes cluster 的痛苦，幫助我們更能專注在開發上。

# 參考連結
 - [AWS 官網文件](https://aws.amazon.com/tw)
 - [kops](https://github.com/kubernetes/kops)

