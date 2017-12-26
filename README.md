# 前言
記得去年看到身邊許多人多參加 [IThome鐵人賽](https://ithelp.ithome.com.tw/ironman)，心中滿滿崇拜，一直期許明年自己也能參加。然而，今年在[IThome鐵人賽](https://ithelp.ithome.com.tw/ironman)報名截止前兩天才赫然想起有這活動XD，雖然參加匆忙、第一次參賽，但仍是希望30天學習筆記不間斷、順利完賽。


# 30天的學習筆記
希望在30天的筆記中，能每天不間斷的跟大家分享，不只帶大家認識Kubernetes，在最後幾天，也能使用[第三方套件Kops](https://github.com/kubernetes/kops)帶著大家操作，在實際應用的環境中，架設Kubernetes與使用。這次的學習筆記可以分為以下幾個方向：

**介紹與開發環境架設**

 - [[[Day 1] 介紹Kubernetes是什麼](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day01)
 - [[Day 2] Minikube 安裝與配置](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day02)
 - [[Day 3] 打造你的docker containers](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03)
 - [[Day 4] 上傳Docker Image到Docker Hub](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day04)
 - [[Day 5] 在Minikube上跑起你的docker containers - Pod](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05)

**Kubernetes基礎概念與實作**

 - [[Day 6] 實際環境運行的Kubernetes - Node](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day06)
 - [[Day 7] 如何擴張我的pods?! - Replication Controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07)
 - [[Day 8] 還在用Replication Controller嗎？不妨考慮Deployments](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day08)
 - [Day 9] 如何讓外部服務與pods的溝通管道 - Services
 - [Day 10] Kubernetes世界不可缺少的 - Labels
 - [Day 11] 如何確保我的程式還在運行 - Healthchecks
 - [Day 12] 敏感的資料怎麼存在k8s?! - Secrets
 - [Day 13] Demo: 架設stateless Wordpress在Kubernetes
 - [Day 14] Kubernetes Web UI介紹

**Kubernetes進階概念與實作**

 - [Day 15] 介紹kops
 - [Day 16] 打造多台機器的cluster - Node Controller
 - [Day 17] Pod之間是如何找到彼此呢 - Service Discovery
 - [Day 18] 把環境參數帶入Kubernetes - ConfigMap
 - [Day 19] Ingress Contoller
 - [Day 20] 如何保存我的資料 - Volumes
 - [Day 21] Volumes Autoprovisoning
 - [Day 22] 在k8s上運行stateful app - Pet Sets
 - [Day 23] 如何監控服務的資源使用 - Monitoring
 - [Day 24] 實現 auto-scaling
 - [Day 25] Demo: 架設stateful Wordpress在Kubernetes


**如何管理Kubernetes**

 - [Day 26] 介紹Kubernetes調度中心 - Master Node
 - [Day 27] 如何限制每個服務資源 - Resource Quotas
 - [Day 28] 如何在k8s管理不同的專案 - Namespaces
 - [Day 29] 如何將一台機器上的服務搬移到另外一台 - Node Maintenance
 - [Day 30] 如何確保k8s的HA - High Avaiability & 總結


# Q&A
筆者也還在學習Kubernetes中，如有對於文章有任何疑問或建議，也歡迎大家來信<zxcvbnius@gmail.com>


# 參考連結
 - [Kubernetes Wikipedia](https://zh.wikipedia.org/wiki/Kubernetes)
 - [GKE 系列文章(一)](https://blog.gcp.expert/kubernetes-gke-introduction/)
 - [kubernetes使用指南](http://www.books.com.tw/products/0010724009)
