# 前言
當 Cluster 上被不同部門或是不同團隊使用時，Kubernetes Cluster 上的資源管理就非常重要了。要如何避免所有資源被單一 container 拿走，要如何根據各個團隊的需求不同提供不同的資源，也是運維人員的一大課題，Kubernetes 提供我們 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 元件讓 Kubernetes 的管理者，不只能限制每個 container 能存取的資源多寡，同時也能透過與 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 的搭配限制每個團隊能使用的總資源。

> 若是還不熟悉 Kubernetes 中的 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 也無須擔心，在明天的學習筆記中將會詳細介紹什麼是 Namespaces。

今天的學習筆記如下：

- 介紹什麼是 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

# 什麼是 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) ？
每一個 container 都可以有屬於它自己的 **resource request** 與 **resource limit**，如同在[昨天的學習筆記中](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day25) 介紹到，我們在設定檔中加入 **spec.resources.requests.cpu** 要求該 container 運行時需要多少 CPU 的資源。`而 Kubernetes 也會透過我們設定的 resource request 去決定要將該 Pod 分配到哪個 Node 上`。

> 我們也可以將 **resource request** 視為該 Pod 最小需要的資源。

除了 resource request 外，[Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 也提供了 **resource limit**，去限制某個 container 最多只能使用的資源。

以 [helloworld-deployment.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day2ˊ/demo-resource-quotas/helloworld-deployment.yaml) 為例，

```
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
          limits:
            cpu: 400m
```

可以發現，在 `spec.resource` 的地方多了一個 `limits` 的欄位，

- **spec.resources.limits.cpu**  
  該欄位代表，這個 container 最多能使用的 cpu 的資源為 `400m 同等於 400milicpu(milicore)`
  
若是透過 `kubectl create` 創建 `hello-deployment`，可以從 [Grafana](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day23) 發現多了 **limit** ，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day26/grafana-get-helloworld-status.png?raw=true)


除了可以針對 **CPU** 與 **Memory** 等計算資源限制之外，也可以限制 
- configmaps 
- persistentvolumesclaims
- pods
- replicationscontrollers
- resourcequotas
- services
- services.loadbalancer
- secrets

等資源**數量上的限制**。

如果我們設置的數量超出指定的數量的範圍，Kubernetes 則會回傳 `403 FORBIDDEN` 提醒我們**超出可使用的配置範圍**。


# 總結
**Resource Qoutas 在實境場景中可以說是一個不可或缺的設定**，避免單一 Pod 使用過多資源而導致其他 Pod 無法正常運行。雖然今天的學習筆記最後沒有時做的部分，但明天的學習筆記中，介紹 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 的同時，也會與使用到 [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)。

# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
