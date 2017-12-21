# [Day 2] Minikube 安裝與配置

# 前言
在介紹完kubernetes之後，筆者想跟大家分享Minikube。



# 什麼是 Minikube

[Minikube](https://github.com/kubernetes/minikube)是由Google發布的一個可以讓Kubernetes在本機上跑的輕量級工具，主要的目的是希望開發者可以透過Mikikube更熟悉kubernetes的指令與環境。Mikikube會在本機上跑起一個virtual machine，並且在這VM裡建立一個signle-node Kubernetes cluster，本身並不支援HA(High availability)，也不推薦在實際應用上運行。

> 對於single-node，或是high availability還不甚熟悉的讀者也無須擔心，在之後的分享都會一一介紹到。




# 安裝 Minikube

Minikube支援 **Windows**、**MacOS**、**Linux**，在這三種平台的本機端都可以安裝並執行Minikube。由於筆者是使用MacOS，所以接下來的安裝步驟，都會以Mac平台平台為主：

- 確認本機端是否已安裝Virtualization Software，像是[VirtualBox](http://virtualbox.org)
- 從[Github](https://github.com/kubernetes/minikube)下載minikube套件
- 手動安裝kubectl套件
- 在minikube上執行hello-minikube app



## 確認本機端是否已安裝Virtualization Software
正如[前言](#前言)提到，minikube會在本機端跑起一個vm，所以在開始安裝minikube之前，需要先確認本機端是否已安裝Virtualization Software。如果還沒有安裝任何VM tool，筆者推薦VirtualBox。

![virtualbox-homepage](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/virtualbox-homepage.png?raw=true)


VirtualBox是一套免費的軟體，支援Windows, MacOS Linux等平台，可以從[官網](http://virtualbox.org)直接下載套件並安裝。

![virtualbox-downloadpage](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/virtualbox-downloadpage.png?raw=true)


安裝完之後會看到歡迎頁面，如[連結](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/virtualbox-welcomepage.png?raw=true)



## 從Github下載minikube套件
在架設好VM環境之後，我們可以直接從minikube的[Github專案](https://github.com/kubernetes/minikube)直接下載套件，以MacOS來說可以直接用`brew`安裝套件

```
$ brew cask install minikube
```

minikube套件約40MB左右。安裝完之後，可以輸入在ternial輸入，`minikube`指令

```
$ minikube
```
![minikube-helppage](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/minikube-helppage.png?raw=true)


安裝完之後，可以用`where指令`查看minikube的安裝位置

```
$ where minikube
/usr/local/bin/minikube
```

也可以查看minikube目前的版本

```
$ minikube version
minikube version: v0.24.1
```

接著啟動minikube

```
$ minikube start
```

若是第一次啟動的讀者，因為要先下載映像檔以及建立VM會花較久的時間

![minikube-start](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/minikube-start.png?raw=true)

其中可以看到 `Kubectl is now configured to use the cluster.`

其實在我們安裝minikube套件的過程中，會順便幫我們安裝好`kubectl`套件。kubectl是kubernetes contoller，之後我們會常常使用它來操作Kubernetes。kubectl是透過 HOME目錄底下的`.kube`目錄的configuration與minikube溝通，可以用`cat`指令查看`~/.kube/config`的內容

```
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
certificate-authority: /Users/{your_username}/.minikube/ca.crt
server: https://192.168.99.101:8443
name: minikube
contexts:
- context:
cluster: minikube
user: minikube
name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
user:
as-user-extra: {}
client-certificate: /Users/{your_username}/.minikube/client.crt
client-key: /Users/{your_username}/.minikube/client.key
```

最後，可以查看minikube是否被正常啟動

```
$ minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.101

```

如果你是安裝virtualbox的話，也可以在VirtualBox介面上看到目前vm的狀態

![minikube-vm-gui](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/minikube-vm-gui.png?raw=true)


## 手動安裝kubectl套件
安裝minikue的時候有可能沒有安裝kubectl的套件，會導致我們在執行`minikube start`指令的時候無法正常啟動。這時我們需要手動安裝kubectl套件：

1) 下載套件(for MacOS)

```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/darwin/amd64/kubectl
```

2) 給予執行權限

```
$ chmod +x ./kubectl
```

3) 將kubectl移到PATH下

```
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

4) 安裝完之後，可以查看kubectl指令

```
$ kubectl
```

![kubectl-introduction](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/kubectl-introduction.png?raw=true)


其他平台可參考[kubectl安裝介紹](https://kubernetes.io/docs/tasks/tools/install-kubectl/)


## 在minikube上執行hello-minikube app
我們可以透過`kubectl run`在minikube上運行一個google提供的hello-minikube docker image，如下指令

```
$ kubectl run hello-minikube --image=gcr.io/google_containers/echoserver:1.8 --port=8080
deployment "hello-minikube" created
```

然後執行`kubectl expose`指令，讓本機端可以連到`hello-minikube`這個服務

```
$ kubectl expose deployment hello-minikube --type=NodePort
service "hello-minikube" exposed
```
接著我們可以使用`minikube service hello-minikube --url`去得到這個service的url

```
$ minikube service hello-minikube --url
http://192.168.99.101:30839
```

打開瀏覽器，貼上url，便可以看到

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day02/hello-minikube-homepage.png?raw=true)

> 1) 每次產生的url是系統決定的
> 2) 可以試著在url後面帶入不同參數，可以看到real path欄位的轉換，可以試試 http://192.168.99.101:30839/hellokube


# 結論
在本機端成功架設minikube之後，接下來兩天會介紹如何在本機端建造的docker image，並上傳到DockerHub。為的是之後介紹kubernetes的功能時，我們將會用自己上傳的docker image來做示範。

# Q&A
歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ^_^

# 參考

- [Minikube Github](https://github.com/kubernetes/minikube)
- [Kubernetes官網](https://kubernetes.io/docs)
