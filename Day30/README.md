# 前言
不知不覺已來到鐵人賽的最後一天。回顧過去這幾十天的筆記，我們可以將這次學習筆記系列大致分成三大部分 

## 基礎 & 進階的 Kubernetes 元件
除了介紹 [Kubernetes](https://kubernetes.io) 基礎的元件，像是 [Pod](https://ithelp.ithome.com.tw/articles/10193232)、[Deployment](https://ithelp.ithome.com.tw/articles/10194152)、[Service](https://ithelp.ithome.com.tw/articles/10194344) 之外，也分享了在 Kubernetes 上較為進階的應用，好比 [Service Discovery](https://ithelp.ithome.com.tw/articles/10195786)、[實現叢集內部負載平衡](https://ithelp.ithome.com.tw/articles/10196261)、以及 [監控叢集的資源使用](https://ithelp.ithome.com.tw/articles/10196938) 等等。希望幫助讀者對 Kubernetes 有基礎了解，日後不論是在日後的實際應用上，或者是想更深入研究時，都能更快速上手。若對 Kubernetes 進階應用還不熟悉的讀者可以回顧以下筆記，

 - [[Day 17] Pod 之間是如何找到彼此呢 - DNS Service Discovery](https://ithelp.ithome.com.tw/articles/10195786)
 - [[Day 18] 高彈性部署 Application - ConfigMap](https://ithelp.ithome.com.tw/articles/10196153)
 - [[Day 19] 在 Kubernetes 中實現負載平衡 - Ingress Controller](https://ithelp.ithome.com.tw/articles/10196261)
 - [[Day 20] 如何保存 Container 中資料 - Volumes](https://ithelp.ithome.com.tw/articles/10196428)
 - [[Day21] 如何動態提供 & 管理儲存資源 - Storage Class & PersistentVolumeClaim](https://ithelp.ithome.com.tw/articles/10196604)
 - [[Day23] 在 Kuberbetes 上實現排程服務 - Cronjob](https://ithelp.ithome.com.tw/articles/10196854)
 - [[Day 24] 如何在 Kubernetes 上監控服務的資源使用 - Heapster](https://ithelp.ithome.com.tw/articles/10196938)
 - [[Day 25] 實現 Horizontal Pod Autoscaling](https://ithelp.ithome.com.tw/articles/10197046)


## Kubernetes 內部的系統運作
在這系列的筆記中，也希望透過簡單的系統介紹，讓讀者不只知道如何操作元件，同時也能對 [Kubernetes](https://kubernetes.io) 其中的運作模式更加了解。像是，當**Kubernetes Cluster 收到一個 request 後，到決定由哪個 Pod 負責回應處理的過程** 抑或是 **輸入一個 kubectl 指令後在 Kubernetes 內部產生的一連串行為** 等等。若對這些有興趣的讀者，不妨回頭參考，
- [[Day 6] 實際環境運行的 Kubernetes - Node](https://ithelp.ithome.com.tw/articles/10193248)
- [[Day 29] 介紹 Kubernetes 調度中心 - Master Node](https://ithelp.ithome.com.tw/articles/10197442)

## 部署產品級的 Kubernetes Cluster
最後但也最重要是，如何將我們**所學的知識真正的運用在實際場景上**。在實現 Kubernetes 第一個會遇到的難題便是`如何架設 Kubernetes Cluster`。以 [kops](https://github.com/kubernetes/kops) 為例，這個第三方套件幫助我們解決很多部署設置上的問題，透過 [kops](https://github.com/kubernetes/kops) 我們只需像操作 kubectl 指令一般，便能幫助我們在 [AWS](https://aws.amazon.com) 上架設一個產品等級的 Cluster，省去了許多環境設置上的問題。有興趣的讀者，也可以看看我們有使用到 [kops](https://github.com/kubernetes/kops) 的筆記
- [[Day 15] 介紹 kops - 在 AWS 上打造 Kubernetes Cluster (上)](https://ithelp.ithome.com.tw/articles/10195575)
- [[Day 16] 介紹 kops - 在 AWS 上打造 Kubernetes Cluster (下)](https://ithelp.ithome.com.tw/articles/10195765)
- [[Day 22] Demo: 在 Kubernetes 上架設 Stateful Wordpress Application](https://ithelp.ithome.com.tw/articles/10196674)

# 延伸閱讀
在短短 30 天中，難以將 [Kubernetes](https://kubernetes.io) 的所有元件與功能都介紹給讀者。除了鼓勵讀者多多閱讀 [Kubernetes 的官方文件](https://kubernetes.io) 之外，筆者也會將日後整理的筆記放在一一在 [Github](https://github.com/zxcvbnius/k8s-30-day-sharing)，若有興趣的讀者也可以持續追蹤。另外，筆者自己本身也還在學習中，若有描述錯的地方也非常歡迎讀者來信告知。

# 感謝
最後，非常感謝在參加鐵人賽期間給予鼓勵與建議的同事與朋友，以及在身邊不斷督促我寫文章的人。如果不是你們很難在沒有任何準備下，第一次參賽還能順利完賽，也許有機會大家明年再見囉 ：）

