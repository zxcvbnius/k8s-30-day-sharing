# 前言
今天的學習筆記將介紹 [Kubernetes](https://kubernetes.io) 另一個元件 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 。[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 協助開發者將一些敏感資訊，像是**資料庫帳密、訪問其他台 server 的 Access Token 、SSH Key**，用 `非明碼的方式(opaque)` 存放在  [Kubernetes](https://kubernetes.io) 中。今天的學習筆記內容如下：

- 介紹什麼是 Secret 
- 實作：Kubernetes 中如何創建 Secret 物件
- 實作：如何掛載 Secret 物件到 Pods 中

> 小提醒：今天的程式碼都可以在 [demo-secrets](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day12/demo-secrets) 上找到。

# 什麼是 Secret ?
正如 [前言]() 提及，[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 是 [Kubernetes](https://kubernetes.io) 提供開發者一種存放敏感資訊的方式。[Kubernetes](https://kubernetes.io) 本身也使用**相同的機制( secrets mechanism) 存放 access token**，限制 API 的存取權限，確保不會有外部服務隨意操作 Kubernetes API。

在 [Kubernetes](https://kubernetes.io) 存取敏感資料(sensitive data)有以下幾種常見的使用方式：

- 將 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 當成 `環境變數(environment variables)` 使用
- 將 [Secrets File](https://kubernetes.io/docs/concepts/configuration/secret/) 掛載(mount) 在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 某個檔案路徑底下使用
- 將這些 sensitive data 統一存放在某一個 Docker Image 中，並將這個 Image 存放在`私有的 Image Registry` 中，透過 `image pull` 下載到 Kubernetes Cluster 中，讓其他 Pods 存取。

今天實作的部分，將針對上述提到的前兩點進行介紹。

# 實作：Kubernetes 中如何創建 Secret 物件
在 [Kubernetes](https://kubernetes.io) 中，創建 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 物件有以下幾種方法：

### 1/ 從檔案匯入 sensitive data
我們可以先將 sensitive data 存在某一個檔案中，透過 `kubectl create ` 指令產生一個 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 物件。  

以帳號密碼為例，我們先將帳號、密碼分別存入兩個不同的檔案，

```
$ echo -n "root" > ./username.txt
$ echo -n "rootpass" > ./password.txt
```

接著使用 `kubectl create secret generic`  指令創建一個 Secret 物件，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kuectl-create-secret-from-file.png?raw=true)

用 `kubectl describe` 查看 **demo-secret-from-file** 物件,

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kubectl-describe-secret-from-file.png?raw=true)  

若用 `kubectl get` 查看所有的 Secrets，

```
$ kubectl get secrets
NAME                    TYPE                                  DATA      AGE
default-token-cljjb     kubernetes.io/service-account-token   3         4m
demo-secret-from-file   Opaque                                2         3m
```

  
會發現除了剛剛創建好的 `demo-secret-from-file`  ，還有一組 `default-token-cljjb` 物件，這是 Kubernetes 內部幫我們建立好的物件。裡面存放著 **一個 token ，開發者可以透過這組 access token 來操控 Kubernetest API**。


### 2/ 從指令輸入 sensitive data
我們也可以透過 `kuectl create` 指令搭配 `--from-literal` 直接在指令後面輸入資料，以上述的帳密為例，

```
$ kubectl create secret generic demo-secret-from-literal \
> --from-literal=username=root \
> --from-literal=password=rootpass

secret "hello-secret-literal" created
```

用 `kubectl describe` 查看 `demo-secret-from-literal` 詳細資料，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kuectl-describe-secret-from-literal.png?raw=true)  


### 3/ 透過 [YAML](https://zh.wikipedia.org/zh-tw/YAML) 創建 [Secret](https://kubernetes.io/docs/concepts/configuration/secret) 物件
我們也可以透過設定檔創建一個 Secret 物件。

首先，須將帳號密碼用 [base64 編碼](https://zh.wikipedia.org/wiki/Base64) ，以 **username=root, password=rootpass** 為例，

```
$ echo -n "root" | base64
cm9vdA==

$ echo -n "rootpass" | base64
cm9vdHBhc3M=
```


並把編碼過的資料寫入 [my-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day12/demo-secrets/my-secret.yaml) 中，

```
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret-from-yaml
type: Opaque
data:
  username: cm9vdA==
  password: cm9vdHBhc3M=
```


透過 `kuectl create` 創建一個新物件，

```
$ kubectl create -f ./my-secret.yaml
secret "demo-secret-from-yaml" created
```

最後可以用 `kubectl get secret` 查看今天創建的三個 Secrets 物件，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kubectl-get-all-secrets.png?raw=true)    

下個章節將透過這些我們創建好的 Secret 物件，介紹如何將這些 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 物件掛載到 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 使用。

# 如何掛載 Secret 物件到 Pods 中
## 1/ 將 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 當成 `環境變數(environment variables)` 使用

以 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day12/demo-secrets/my-pod.yaml) 為例，

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: webserver
spec:
  containers:
  - name: demo-pod
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: demo-secret-from-yaml
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: demo-secret-from-yaml
          key: password
```

在 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day12/demo-secrets/my-pod.yaml) 中，我們設定 `env` 會去從指定的 [Secret 物件](https://kubernetes.io/docs/concepts/configuration/secret/) 找出相對應的值。值得一提的是，**若是該 Secret 物件是從 `--from-file` 創建，那麼 Kubernetes 會把檔案名稱當成 key ，檔案內容當成 value**。

使用 `kubectl create` 創建 `my-pod` ，指令如下

```
$ kubectl create -f ./my-pod.yaml
pod "my-pod" created
```

當 `my-pod` 建立好時，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kubectl-get-pod-status.png?raw=true)    

我們可以透過 `kubectl exec` 進到 `my-pod` 裡面，

```
$ kubectl exec -it my-pod -- /bin/bash
```

查看 **SECRET_USERNAME** 與 **SECRET_PASSWORD**，如我們在 `demo-secret-from-yaml` 設定一致，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kubectl-set-secret-as-env.png?raw=true)    


## 2/ 將 [Secrets File](https://kubernetes.io/docs/concepts/configuration/secret/) 掛載(mount) 在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 某個檔案路徑底下使用

我們也可以將 secrets 掛載到，Pod 底下的某個路徑檔案，以 [my-pod-with-mounting-secret.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day12/demo-secrets/my-pod-with-mounting-secret.yaml) 為例，

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-with-mounting-secret
  labels:
    app: webserver
spec:
  containers:
  - name: demo-pod
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: demo-secret-from-yaml
```


如同 創建 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day12/demo-secrets/my-pod.yaml) 一樣，使用 `kubectl create` 指令創建一個新的 Pod 物件，

```
$ kubectl create -f ./my-pod-with-mounting-secret.yaml
pod "my-pod-with-mounting-secret" created
```

透過 `kubectl exec` 進入 `my-pod-with-mounting-secret` 這個物件，且可以在我們指定掛載的 **/etc/creds** 找到存在 [demo-secret-from-yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day12/demo-secrets/demo-secret-from-yaml.yaml) 中的資料

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day12/kubectl-mount-demo.png?raw=true)    

透過以上兩種方式，我們便能在 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 中存取 Secret 物件。

# 總結
當我們有些敏感資料需要傳入 Kubernetes 時，使用 [Secret](https://kubernetes.io/docs/concepts/configuration/secret) 是個不錯的選擇 。然而，需要留意的是，一旦建立 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 物件，其他人可以也可以在 Kubernetes Cluster 上存取 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 中的敏感資料。因此，我們需要搭配 [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 來限制其他人的存取權限，在之後 [[Day 28] 如何在 k8s 管理不同的專案 - Namespaces](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day28) 也會與大家分享如何設置。我們明年見囉～ :tada:


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )





# 參考連結

 - [Kubernetes官網](https://kubernetes.io)

# 勘誤
感謝網友 @herb123456 提醒，base64 是一種編碼方式，非加密方式。
