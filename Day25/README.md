# 前言
在昨天的學習筆記中介紹到如何[監控 Kubernetes 上的資源使用](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day24) 後，今天筆者想跟大家分享如何在 [Kubernetes](https://kubernetes.io) 上實現 [Autosacling](https://en.wikipedia.org/wiki/Autoscaling)。在許多實際場景中，應用服務常常需因應不同流量而配置不同的資源。好比：原本的應用服務可能每天都只有 10 人使用，我們只需要架設一台小 server 即可應付這些流量；當有天應用服務因為某些因素使用人數上升到 10 萬人，可能我們原有的資源不足以回應這些流量，導致使用者無法連上該應用服務。如果這時候，系統能幫我們`根據目前的資源使用率，決定是否自動調整資源(像是加開 server)來回應這些流量就太好了`。幸好，這樣的功能 [Kubernetes](https://kubernetes.io) 也幫我們想到了。

今天的學習筆記將介紹，

- 什麼是 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- 實作：在 [minikube](https://github.com/kubernetes/minikube) 中實現 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

# 什麼是 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
如[前言]()所提，不論身為運維人員或是開發者，都希望系統能根據目前的資源使用率，決定是否自動調整資源來回應這些流量，避免服務無法運行而失去使用者。在目前許多第三方雲端服務供應商也都提供 [Autosacling](https://en.wikipedia.org/wiki/Autoscaling) 的功能，像是 [AWS](https://aws.amazon.com/tw/autoscaling/) 或是 [Google Cloud Platfrom](https://cloud.google.com/compute/docs/autoscaler/)。而 Kubernetes 本身也提供一個 API - [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)，讓我們可以針對不同的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 的使用量，來決定是否新增或減少 Pod 的數量。 

以下以 [helloworld-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/demo-hpa/helloworld-depolyment.yaml) 與 [helloworld-hpa.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/demo-hpa/helloworld-hpa.yaml) 為例，

```
# helloworld-deployment.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: helloworld-pod
  template:
    metadata:
      labels:
        app: helloworld-pod
    spec:
      containers:
      - name: my-pod
        image: zxcvbnius/docker-demo:latest
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: 200m
```

可以發現，在 `spec` 的地方多了一個 `resources` 的欄位，

- **spec.resources.requests.cpu**  
  透過該欄位的設置，代表當 Kubernetes 在運行該 Pod 時，需要配置 `200m` CPU 給該 Pod。`200m 同等於 200milicpu(milicore)`，代表要求一個 CPU 20% 的資源。然而直得注意的是，若該 Node 是多核(multiple cores)，則 `200m` 代表要求使用每一核(core)百分之二十的資源。更多關於 `spec.resources.requests` 可以參考該[連結](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container)

在看完，[helloworld-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/demo-hpa/helloworld-depolyment.yaml) 後，來看看 [helloworld-hpa.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/demo-hpa/helloworld-hpa.yaml)，

```
# helloworld-hpa.yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: helloworld-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta2
    kind: Deployment
    name: helloworld-deployment
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

- **spec.scaleTargetRef**  
  指定 autoscaling 的對象
  
- **spec.targetCPUUtilizationPercentage**  
  以 `helloworld-deployment` 為例，我們指定 `CPU 200m 的資源`，代表**當該 helloworld-pod CPU 使用率達到 100 m 時，HorizontalPodAutoscaler 就會幫我們新產生一個 Pod**。

## 支持 Colddown / Delay
[Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 也支援 Colddown 與 Delay，避免 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 一次產生太多 Pod 或是關掉太多 Pod。在建立 Kubernetes Cluster 時，可以加入 `--horizontal-pod-autoscaler-downscale-delay` 與 `--horizontal-pod-autoscaler-upscale-delay` 去限制 Autoscaling 的回應時間。

- **--horizontal-pod-autoscaler-downscale-delay**  
  代表 autoscaling 須等多久的時間才能進行 downscale，預設為 `5 分鐘`。
  
- **--horizontal-pod-autoscaler-upscale-delay**    
  代表 autoscaling 須等多久的時間才能進行 upscale，預設為 `3 分鐘`。  
  


# 實作：在 [minikube](https://github.com/kubernetes/minikube) 中實現 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

> 需要注意的是，使用 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 之前，[minikube](https://github.com/kubernetes/minikube) 必須先安裝好 [heapster](https://github.com/kubernetes/heapster) ，讓 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 可以知道目前 Kubernetes Cluster 中的資源使用狀況(metrics)。若還是不知道怎麼在 Kubernetes 安裝 [heapster](https://github.com/kubernetes/heapster) 的讀者，不妨參考[昨日的學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day24)。

在了解 [helloworld-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/demo-hpa/helloworld-depolyment.yaml) 與 [helloworld-hpa.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/demo-hpa/helloworld-hpa.yaml) 後，透過 `kubectl create` 創建 `helloworld-deployment`，指令如下，

```
$ kubectl create -f ./helloworld-depolyment.yaml
deployment "hello-deployment" created
```

接著創建一個 `helloworld-service` ，讓 Kubernetes Cluster 中的其他物件可以訪問到 `helloworld-deployment`，指令如下：

```
$ kubectl expose deploy helloworld-deployment \
> --name helloworld-service \
> --type=ClusterIP
service "helloworld-service" exposed
```

用 `kubectl get` 查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/kubectl-get-all-2.png?raw=true&version=2)

接著創建 `helloworld-hpa`，指令如下

```
$ kubectl create -f ./helloworld-hpa.yaml
horizontalpodautoscaler "helloworld-hpa" created
```

用 `kubectl get` 檢視目前狀況，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/kubectl-get-hpa.png?raw=true)

可以看到 `TARGETS` 的欄位目前是，`<unknown> / 50%`。稍等久一點，等 `helloworld-hpa` 從 `heapster` 抓到目前的資料則可看到數字的變化，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/kubectl-get-hpa-2.png?raw=true)

從 [Grafana](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day24) 也可以觀察到 `helloworld-deployment` 的狀態，正如我們在 YAML 設定檔要求 200 milicores，目前的使用率為 0%

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/grafana-helloworld-usage.png?raw=true)


接著，我們要在 minikube 裡面運行一個 server **不斷去訪問 `helloworld-pod` 使 CPU 的使用率超過 100 m，再來觀察 helloworld-hpa 是否會偵測到幫我們新增 Pod**，指令如下：

```
$ kubectl run -i --tty alpine --image=alpine --restart=Never -- sh
```

> 若是忘了上述指令的讀者，不妨回頭看看 [[Day 5] 在 Minikube 上跑起你的 Docker Containers - Pod & kubectl 常用指令](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05) 裡面有介紹到該指令的功用唷。

接著安裝 `curl` 套件，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/alpine-install-curl.png?raw=true)

接著訪問 `helloworld-service`，會吐回 `Hello World!` 的字串，

```
$ curl http://10.108.56.58:3000
Hello World!
```

接著，我們設置一個無窮迴圈，透過 curl 不斷送請求給 `helloworld-deployment`，指令如下，

```
$ while true; do curl http://10.108.56.58:3000; done
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/curl-while-loop.png?raw=true)

接著，再回頭看 `helloworld-hpa` 的狀態，可以發現目前 CPU 的使用率以超出我們所設定的 50%，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/kubectl-get-hpa-autoscaling-1.png?raw=true)

若再觀察一陣子，會看到 `helloworld-deployment`底下已有 4 個 Pod，代表 `helloworld-hpa` 幫我們實現了 Autoscaling

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/kubectl-get-hpa-autoscaling-2.png?raw=true)

我們停止 `curl` 指令，過了一陣子之後可以發現原本 4 個 Pod 已經退回到原本設定的 2 個了

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/kubectl-get-hpa-autoscaling-3.png?raw=true)

若從 [Grafana](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day24) 也可以看到剛剛的變化，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day25/grafana-helloworld-usage-2.png?raw=true)

# 總結
[Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 可以說是 Kubernetes 中一個不可或缺的 API，透過 Kubernetes 自動化的管理更增加了應用服務的可用性。在到今天為止的學習筆記中，我們介紹了**基礎 & 進階 的 Kubernetes API**，希望讀者在看完之後都能有所收穫。在學習筆記的最後幾天，我們將`從管理層面介紹 Kubernetes`，讓讀者更能掌握 Kubernetes 在實際場景中的運用。

# Q&A
最後，歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
