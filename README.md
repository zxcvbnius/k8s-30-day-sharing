# 前言

第一次認識 **Kubernetes(aka. k8s)** ，約莫去年夏天時候的事。那時還在一家新創擔任後端工程師，在資源有限的情況下，每個人都須身兼多職，除了開會、討論產品、程式開發、有時還需拜訪客戶。而同時，系統維護以及產品更新也是我們一個需要顧及的課題，如何快速開發客戶的需求、如何確保系統的穩定、以及如何協助團隊之間合作更流暢等，可以說是忙得焦頭爛額。而 Kubernetes 給人最直接的感受是，相較於系統複雜的設定，只需要一名系統兼運維的工程師負責部署與維護、其他人就能更專注在開發上，**即便像我們這樣『小』團隊、也是有能力面對複雜的系統設計，這並不是因為我們做了什麼，而是 Kubernetes 已經幫我們做了很多事情**。 


# 未來 30 天的學習筆記
希望在未來 30 天裡，能每天不間斷的跟大家分享，不只帶大家認識 Kubernetes ，在最後幾天，也能使用 [第三方套件Kops](https://github.com/kubernetes/kops) 帶著大家操作，在實際應用的環境中，架設 Kubernetes 與使用。這次的學習筆記可以分為以下幾個方向：


**介紹與開發環境架設**

 - [[Day 1] 介紹 Kubernetes 是什麼](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day01)
 - [[Day 2] Minikube 安裝與配置](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day02)
 - [[Day 3] 打造你的 Docker Containers](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03)
 - [[Day 4] 上傳 Docker Image 到 Docker Hub](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day04)
 - [[Day 5] 在 Minikube 上跑起你的 Docker Containers - Pod & kubectl 常用指令](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day05)

**Kubernetes基礎概念與實作**

 - [[Day 6] 實際環境運行的 Kubernetes - Node](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day06) 
 - [[Day 7] 如何擴張我的 Pods ?! - Replication Controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07)
 - [[Day 8] 還在用 Replication Controller 嗎？不妨考慮 Deployments](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day08)
 - [[Day 9] 建立外部服務與 Pods 的溝通管道 - Services](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day09)
 - [[Day 10] Kubernetes 世界不可缺少的 - Labels](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day10)
 - [[Day 11] 如何確保 Container 運行狀態 - Health Checks](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day11)
 - [[Day 12] 敏感的資料怎麼存在k8s?! - Secrets](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day12)
 - [[Day 13] Demo: 架設 Stateless Wordpress 在 Kubernetes](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13)
 - [[Day 14] Kubernetes Dashboard 介紹](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day14)

**Kubernetes進階概念與實作**

 - [[Day 15] 介紹 kops - 在 AWS 上打造 Kubernetes Cluster (上)](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day15)
 - [[Day 16] 介紹 kops - 在 AWS 上打造 Kubernetes Cluster (下)](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day16)
 - [[Day 17] Pod 之間是如何找到彼此呢 - DNS Service Discovery](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day17)
 - [[Day 18] 高彈性部署 Application - ConfigMap](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day18)
 - [[Day 19] 在 Kubernetes 中實現負載平衡 - Ingress Controller](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day19)
 - [[Day 20] 如何保存 Container 中資料 - Volumes](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day20)
 - [[Day21] 如何動態提供 & 管理儲存資源 - Storage Class & PersistentVolumeClaim](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day21)
 - [[Day 22] Demo: 在 Kubernetes 上架設 Stateful Wordpress Application](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day22)
 - [[Day23] 在 Kuberbetes 上實現排程服務 - Cronjob](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day23)
 - [[Day 24] 如何在 Kubernetes 上監控服務的資源使用 - Heapster](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day24)
 - [[Day 25] 實現 Horizontal Pod Autoscaling](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day25)


**如何管理Kubernetes**

 - [[Day 26] 如何管理 Kubernetes 上的資源分配 - Resource Quotas](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day26)
 - [[Day 27] 在 Kubernetes 上管理不同的專案 - Namespaces](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day27)
 - [[Day 28] 如何將一台機器上的服務搬移到另外一台 - Node Maintenance](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day28)
 - [[Day 29] 介紹 Kubernetes 調度中心 - Master Node](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day29)
 - [[Day 30] Kubernetes 30天學習總結](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day30)




# 所以，什麼是Kubernetes

![Kubernetes](https://kubernetes.io/images/favicon.png)  
(圖片擷取自：[kubernetes.io](https://kubernetes.io/images/favicon.png))



> 如果對於Docker, Container還不太熟悉的讀者，不妨先看過IThome的專欄 [Container技術三部曲](https://www.ithome.com.tw/article/91838) ，也許會對什麼是container更加了解

[Kubernetes](https://kubernetes.io/) 是一個協助我們自動化部署、擴張以及管理容器應用程式(containerized applications)的系統。相較於需要手動部署每個容器化應用程式(containers)到每台機器上，Kubernetes 可以幫我們做到以下幾件事情：  

 - 同時部署多個 containers 到一台機器上，甚至多台機器。  
 - 管理各個 container 的狀態。如果提供某個服務的 container 不小心 crash 了，Kubernetes 會偵測到並重啟這個 container，確保持續提供服務  
 - 將一台機器上所有的 containers 轉移到另外一台機器上。  
 - 提供機器高度擴張性。Kubernetes cluster 可以從一台機器，延展到多台機器共同運行。    
 
 
 
# 為何使用Kubenetes
筆者過去曾參與過大型專案開發，上線產品、除錯、與測試功能都包在一起。每次發布新功能、修改代碼都非常膽戰心驚，哪怕是一個 bug 也會影響整個系統效能。而相較於這樣 [單體架構(Monolithic Architecture)](https://www.nginx.com/blog/refactoring-a-monolith-into-microservices/) 的服務， [微服務(microservices)架構](https://www.nginx.com/blog/introduction-to-microservices/) 大大減少程式複雜度，將每個服務依照各自業務需求獨立出來，以 Rest API 互相構通。而 microservices 概念的導入，改善了我們過去所面臨到的問題：

 - 將龐大的專案拆成幾個不同面向的小專案，當代碼夠小、容易理解、開發效率能被提高
 - 各個服務之間也可獨立部署，不因一個服務癱瘓而癱瘓整個系統
 - 各團隊可以依照自己的需求使用適合自己的語言、資料庫開發
 - 每個服務也可以依照自己的需求，選擇在不同機器上部署

然而，當系統中的微服務越來越多時，管理上也會面臨到很大的挑戰。Kubenetes 的出現讓我們管理這些微服務程式更加方便。



# Kubernetes 的優點
 - **可以跑在任何地方 Can run anywhere**   
	Kubernetes 可以運行在任何地方：不論是私有雲、公有雲(像是 AWS, Google Cloud Platform )、或是混合雲。
	
 - **高度模組化 High modular**  
   每個服務都被切成一個 container ，不論是要做修改、擴張、甚至將服務遷移到另外一台機器，都可以快速被部署。
   
 - **活躍的社群 Open source & active community**   
   Kubernetes 是 [開源的](https://github.com/kubernetes/kubernetes) ，受到社群的關注度也非常高。

 - **Google的背書 Backed by Google**  
   最初版的 Kubernetes 是由 Google 內部 Borg team 的成員撰寫且現在仍在持續維護。Google 使用他們自身的系統 Borg 管理容器化應用長達十年多。Kubernetes 的目的即是將 Borg 最精華的部分取出來，使得開發者能夠更簡單、直接應用。



# Q&A
筆者也還在學習Kubernetes中，如有對於文章有任何疑問或建議，也歡迎大家留言給我唷 : )



# 參考連結
 - [Kubernetes Wikipedia](https://zh.wikipedia.org/wiki/Kubernetes)
 - [GKE 系列文章(一)](https://blog.gcp.expert/kubernetes-gke-introduction/)
 - [kubernetes 使用指南](http://www.books.com.tw/products/0010724009)
