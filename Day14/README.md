# 前言
在前幾天的學習筆記中，我們都是透過 [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) 指令來操作 [Kubernetes](https://kubernetes.io) 。筆者今天想分享另外一個由 [Kubernetes](https://kubernetes.io) 提供的操作介面 - [Dashboard](https://github.com/kubernetes/dashboard)。透過 [Dashboard](https://github.com/kubernetes/dashboard) 提供的圖形介面，開發者能更快速、方便地查看 Kubernetes Cluster 上資源分佈與使用狀況。

今天的學習筆記內容下：

- 如何安裝 [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)
- Kubernetes Dashboard 功能介紹

# 安裝 Kubernetes Dashboard

## Minikube
如果是在本機端使用 `minikube` 的讀者，可以在終端機輸入 `minikube dashboard` ，如此在本機端的瀏覽器就會自動開啟 Dashboard 頁面，

```
$ minikube dashboard
```

如果只是想取得 Dashboard 的 url ，可以在指令後面加上 `--url`

```
$ minikube dashboard --url
http://192.168.99.100:30000
```

## General Case
### 安裝 Dashboard UI
一般 Kubernetes 預設並不會有 Dashboard 套件，因此我們需要透過以下的指令，透過 `kubectl apply` 指令創建這個 Web UI component，

```
$ kubectl apply -f https://raw.githubusercontent.com/\
> kubernetes/dashboard/master/src/deploy/recommended/\
> kubernetes-dashboard.yaml
```

### 透過 kubectl proxy 連接 
安裝完 Dashboard 之後，我們可以透過 `kubectl proxy` 指令連接到 Dashboard，預設會將 Dashboard 與本機端的 `port number 8001` 互相 mapping，而開發者可以在本機端上直接透過[該 url](http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard:/proxy/) 存取 Dashboard。

```
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/kubectl-proxy-demo.png?version=v2)

值得留意的是
 - kubectl proxy 本身提供許多 options 讓該指令的應用更加彈性。例如，若是我們希望別台機器也能透過**下 kubectl proxy 指令的機器存取 Dashboard**，則我們可以在指令後面加上 **--address** 與 **--accept-hosts** 如以下指令，
```
$ kubectl proxy --address='0.0.0.0' --port=8002 --accept-hosts='^*$'
```


 - 目前官方已不推薦使用 [http://localhost:8001/ui](http://localhost:8001/ui) 存取 Dashboard，而是用 [上圖中顯示的連結](http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard:/proxy/#!/overview?namespace=default)


 - 更多關於 Dashboard 透過 kubectl proxy 存取的使用，可以參考：  
   [Kubernetes Dashboard README](https://github.com/kubernetes/dashboard) 與 [FAQ](https://github.com/kubernetes/dashboard/wiki/FAQ)

### 透過 Master ApiServer 連接
由於透過 `kubectl proxy` 連到 Dashboard 沒有認證機制，可能會有安全上的疑慮。在日後的學習筆記中，會與大家分享如何透過帳密，從 [Kubernetes master apiserver]() 存取 Kubernetes Dashboard。

# Kubernetes Dashboard 功能介紹
[Kubernetes Dashboard](https://github.com/kubernetes/dashboard) 能替我們做到以下幾件事情：

- 透過 Web UI 能快速瀏覽目前運行在 Kubernetes 上所有的 applications

- 在 Dashboard 上，可以直接 **新增、修改、刪除** 物件 (如同 kuberctl create、 edit 、 delete 指令)

- 可以取得每個物件的歷史狀態 (如同 kubectl describe pod )

在今天學習筆記中，我們將在本機端透過 `minikube`  指令，訪問 Kubernetes Dashboard，指令如下：

```
$ minikube dashboard
```


![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/minikube-dashboard-overview.png)

Dashboard 首頁會顯示目前所有物件的工作的狀態，這裡我們可以看到目前運行的是[我們昨天架設的 wordpress application](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13) 。

點選 `wordpress-app` 物件後，我們可以看到該物件的詳細資料，包含創建的時間、使用的 Lables 等等。

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/minikube-dashboard-deploy-detail-page.png)

而這樣的查看功能，也適用於每一個先前我們介紹過的 Kubernetes 元件。此外，我們也可以透過這個 Web UI 創建一個新的物件。找到右上角的 **+CREATE** 選項，進入編輯頁面之後，可以上傳預先寫好的 YAML 設定檔，或是直接在上面編輯。

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/minikube-dashboard-create-obj.png)

另外，我們也可以在 Dashboard 刪除物件，以 `wordpress-app` Pod物件為例，進到 Pod 的詳細頁面之後，右上角點選 DELETE 選項，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/minikube-dashboard-delete-obj.png)

會跳出一個確認的對話視窗，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/minikube-dashboard-delete-obj-confirm-dialog.png?version=1)

按下 **YES** 之後，該 Pod 物件就會被刪除。若這時我們再回到 Overview 頁面，會發現 Kubernetes 會幫我們建立一個新的物件。


![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day14/minikube-dashboard-before-deleting-obj.png?version=1)

這是因為 `wordpress-deployment` 偵測到 Pod 數量不符合預期，因此新創建一個 Pod 維持外部服務能存取 Wordpress Application。

> 可以看到右上角多了幾個選項，**EXEC**，**LOGS**，**EDIT**，**DELETE**。每個選項都與我們先前介紹的 kubectl 指令相仿，有興趣的讀者不妨在 Dashoard 上操作看看這些功能。

# 總結
雖然今天對 [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) 的介紹不太多，但相信已經熟悉 [kubectl 指令](https://kubernetes.io/docs/reference/kubectl/overview/) 的讀者很快就能上手。在之後的學習筆記中，我們將繼續透過 [kubectl 指令](https://kubernetes.io/docs/reference/kubectl/overview/) 分享更多 Kubernetes 的元件。

# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 : )



# 參考連結
 - [Kubernetes 官網](https://kubernetes.io)
 - [Kubernetes Dashboard Github](https://github.com/kubernetes/dashboard)
