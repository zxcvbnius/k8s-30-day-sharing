# [Day 11] 如何確保我的程式還在運行 - Health Checks

# 前言
正如前幾天提及 [Kubernetes](https://kubernetes.io) 可以偵測到 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 的生命週期去調整 Kubernetes Cluster 中其他物件的狀態。然而有些時候，**雖然 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 還在運行，但在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 中的 web app container 可能因為某些原因已經停止運作，或是資源被其他 containers 佔用，導致我們送去的 request 無法正常回應。**幸好，[Kubernetes](https://kubernetes.io) 也幫我們想到這點了，它提供 [Health Checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) 協助我們去偵測 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 中的 containers 是否都還正常運作，確保服務本身也能正常運行。

今天的學習筆記內容如下：

- 在 Kubernetes 中的 Health check 有哪些 ？
- 實作：如何在 Pod 中加入 Health check

> 小提醒：今天的程式碼都可以在 [demo-health-checks](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day11/demo-health-checks) 上找到。



# 在 Kubernetes 中的 Health Checks 有哪些 ？

在 [Kubernetes](https://kubernetes.io) ，有兩種常見的 health checks，

- 定期的`透過指令`去訪問 container
- 定期`發送一個 HTTP request` 給 container

若當我們設定的 health checks 偵測到異常時，Kubernetes 會 restart container，確保應用服務可正常運行(avaiable)。

在今天的實作中，我們將使用 [my-deployment-with-health-checks.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day11/demo-health-checks/my-deployment-with-health-checks.yaml)  來創建一個 Deployment 物件，並透過`定期發送一個 HTTP request 給 container` 的方式，來判斷目前 web app container 是否還正常運作。

# 實作：如何在 Pod 中加入 Health check

[my-deployment-with-health-checks.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day11/demo-health-checks/my-deployment-with-health-checks.yaml) 內容如下，

```
apiVersion: apps/v1beta2 # for kubectl versions >= 1.9.0 use apps/v1
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
        image: zxcvbnius/docker-demo
        ports:
        - name: webapp-port
          containerPort: 3000
        livenessProbe:
          httpGet:
            path: /
            port: webapp-port
          initialDelaySeconds: 15
          periodSeconds: 15
          timeoutSeconds: 30  
          successThreshold: 1
          failureThreshold: 3
```


可以看到在  [my-deployment-with-health-checks.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day11/demo-health-checks/my-deployment-with-health-checks.yaml) 中多了 `livenessProbe` 設定。

### livenessProbe
在 [livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) 中我們可以做以下設定，

- **httpGet.path**   
  設定  health checks 要訪問的路徑

- **httpGet.port**  
  指定我們要訪問的 port，這裡 port number 是 3000

- **initialDelaySeconds**    
  設定當 service 剛啟動時，要延遲幾秒再開始做 health check

- **periodSeconds**     
  代表每隔幾秒訪問一次，預設值為 `10秒`

- **successThreshold**    
  可以設置訪問幾次就代表目前 service 還正常運行

- **failureThreshold**    
  代表 service 回傳不如預期時，在 Kubernetes 放棄該 container 之前，會嘗試的次數，預設為`3`次。


> 通常在實際場景中(Production Environment)，我們會有一個專門來回應 health check 的 endpoint，例如 `/health` 來確認 application 是否正常運作。

在暸解 `livenessProbe` 的各個設定之後，我們可以在終端機輸入 `kubectl create` 指令，創建一個名為 `hello-deployment` 的物件。

```
$ kubectl create -f ./my-deployment-with-health-checks.yaml
deployment "hello-deployment" created
```

接著，透過 `kubectl get` 來查看 `hello-deployment` ，

```
$ kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-deployment   3         3         3            3           23s
```

以及 `Pods` 的狀態 ，

![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day11/kubectl-get-hello-deployment-pods.png?raw=true)

我們可以使用 `kubectl describe` 看其中一個 Pod 的詳細資料，以 `hello-deployment-6b56dd6494-mx5jk` 為例


```
$ kubectl describe pod hello-deployment-6b56dd6494-mx5jk
```


![](https://raw.githubusercontent.com/zxcvbnius/k8s-30-day-sharing/master/Day11/kubectl-describe-pod.png?raw=true)


可以在 `Liveness` 的欄位裡面，清楚看到目前 health check 的設定狀態。當 container 無法正常回應 health check 時，Kubernetes 就會視為該 container 失去功能並重啟。

# 總結
雖然今日的學習筆記只有分享，如何設置 `定期訪問的 http reqeust` ，然而 `使用指令或 TCP 定期訪問服務` 也是 health check 常見的方式。若有興趣的讀者，不妨參考 [官網提供的範例](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) ，相信也會有所收穫！
