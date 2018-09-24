# 前言
不知讀者是否還有印象 [前一天學習筆記](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day20) 最後提到，當我們希望將 [AWS EBS](https://aws.amazon.com/tw/ebs) 掛載在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 的某個指定路徑之前，我們須先手動輸入 `awscli` 指令，創建一個 EBS，指定自己要的**型態/type**、**空間大小/size**、以及指定 **該 EBS 的所在地/Region** ，並取得該 EBS 的 **VolumeID** 後，再透過 YAML 設定檔掛載在 Pod 指定的路徑底下。

如果今天我們只有一兩個 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 有掛載 Volumes 的需求，這些指令看似沒什麼。但當今天需要`處理數以千計的 Pod 且依照每個 Pod 不同的需求，還需掛載不同型態、不同大小、不同所在地的 Volumes 時`，這樣的步驟似乎就顯得擾人。不但不好管理，且當 Pod 不小心掛掉後，或是我們將某應用服務(Pod)收起來後，我們還需 **額外管理、手動刪除沒在使用的 Volumes**。幸好，[Kubernetes](https://kubernetes.io) 提供了我們另一個元件 - [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes) ，不只能幫我們`定義每個 Volumes 物件規格`，還能因應當下需求而`動態產生`相對應的 Volumes。Kubernetes 這樣貼心的設計，也使得我們可以統一查看、管理這些 Volumes 的使用狀態。

> 以 [AWS EBS](https://aws.amazon.com/tw/ebs) 為例，當 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 停止運行或該 EBS 不再掛載到任何服務上時，Kubernetes 還能幫我們自動從 AWS 上銷毀該物件，避免被收取不必要的費用，大大提升 Volume 的彈性與可用性。

今天的學習筆記中，將介紹幾個新物件：
- [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes)
- [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
- 如何將動態產生的 Volume 掛載在特定 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 中

# [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes)
類似於程式語言中 `類別(Class)` 的概念，透過 Kubernetes 提供的 [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes) 元件，我們可以依據需求，根據 Volumes 的**提供者(provisioner)、類型(type)、所在地(Region)，以及回收政策(reclaimPolicy)去定義不同的 Storage Class**,以 [ebs-standard-storage-class.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day21/demo-storage-class/ebs-standard-storage-class.yaml) 為例，

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zone: us-west-2
reclaimPolicy: Delete
```

 - **apiVersion**  
   目前 api 版號為 `storage.k8s.io/v1` 
 - **metadata.name**  
   定義該 SotrageClass 物件的名稱
 - **provisioner**     
   在這個設定檔中，我們希望儲存空間使用 AWS 的 [EBS](https://aws.amazon.com/tw/ebs) 服務
 - **parameters.type**     
   在該欄位中，可以定義 EBS 的 的種類，若 `provisioner` 為 `kubernetes.io/aws-ebs` 但沒有設定該值的話，預設皆為 **gps**。更多關於 [AWS EBS](https://aws.amazon.com/tw/ebs) 有支援的種類可以參考 [該連結](https://aws.amazon.com/tw/ebs/pricing/)
 - **parameters.zone**     
   代表我們希望該 EBS 可以放置在 AWS 的 `us-west-2(Oregon)` 這個 region。值得特別注意的是，**AWS 的 EBS 只能供 EC2 使用，且 EBS 的所在地必須與 EC2 相同**。
 - **reclaimPolicy**       
**是指由該 Storage Class 作為模板產出的 Volumes ，在綁定的 Pod 消失後的行為**。Kubernetes 提供兩種型別，`Delete` 與 `Retain`。`Delete` 代表當綁定的 Pod 消失後，該 Volume 相對應得資源(ex. EBS)也會自動移除，我們無需再額外下指令操作刪除；**Retain 的行為與直接在 Pod 定義檔中指定 Volume 相同**，則是只即便 Pod 消失後，Volume 相對應的物件資源並不會跟著消失。StorageClass 預設的行為為`Delete`。
   
> 此外，[Kubernetes 的官網也列出了所有支援 Storage Class 的供應商 (Provisioner)](https://kubernetes.io/docs/concepts/storage/storage-classes/#mount-options)，有興趣的讀者不妨可以看看除了 AWSElasticBlockStore plugin 以外，Kubernetes 還提供哪些類型的 plugin。

# [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
[PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 可以透過我們設定好的 Storage Class 的模板，創建出我們所需要的資源。不再需要預先創建好外部儲存資源(例如：已架好的 NFS，以申請可用的 AWS EBS 或是 [GCE Persistent Disk](https://cloud.google.com/compute/docs/disks/))，而是可以根據 Kubernetes Cluster 內部的需求，**動態產生**相對應的儲存區塊，並與 Cluster 中其他的物件的物件綁定(bind)使用。在解除綁定後，[PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 也可以根據 Storage Class 設定的`回收機制(Reclaim Policy)`決定是否要保存該 Volume 或是刪除。

以 [my-persistent-volume-claim.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day21/demo-storage-class/my-persistent-volume-claim.yaml) 為例，

```
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
```

 - **apiVersion**  
   目前 api 版號為 `storage.k8s.io/v1` 
 - **metadata.name**  
   定義該 PersistentVolumeClaim 物件的名稱
 - **spec.accessModes**     
   Kubernetes 提供三種 [Access Modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)，分別為`ReadWriteOnce`，`ReadOnlyMany`，`ReadWriteMany`，用途如下：
   
    - **ReadWriteOnce**  
      該 PersistentVolumeClaim 產生出來的 Volume 同時只可以掛載在同一個 Node 上提供讀寫功能。
      
    - **ReadOnlyMany**        
      該 PersistentVolumeClaim 產生出來的 Volume 同時可以在多個 Node 上提供讀取功能
      
    - **ReadWriteMany**              
      該 PersistentVolumeClaim 產生出來的 Volume 同時可以在多個 Node 上提供讀寫功能
      
> 然而需要注意的是，不是每種 [Volume Plugin](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner) 都提供這些功能。以 [AWS EBS](https://aws.amazon.com/tw/ebs) 為例，AWS EBS 只支援 ReadWriteOnce，只允許該 Volume 同時間只能供一個 Node 讀寫。[Kubernetes 的官網上也列出所有不同類型的 Volume Plugin 支援的模式](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)，讀者在選擇 Volume Plugin 時，需注意該 Volume Plugin 能否滿足應用端的需求。

- **spec.resources.requestes.storage**   
  代表我們請求的儲存空間大小為 8 GB

- **spec.storageClassName**  
  指定我們選擇想使用哪個 Storage Class 作為模板。在這個設定檔中，我們指定剛剛創建名稱為 `standard` 的 Storage Class。如此，Kubernetes 會幫我們在 AWS 中創建一個 8G 大小且位於 Oregon 的 EBS。

# 如何將動態產生的 Volume 掛載在特定 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 中
最後，在設定好 PersistentVolumeClaim 格式後，我們可以將這個動態產生的 PersistentVolumeClaim 與我們指定的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 互相綁定，以 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day21/demo-storage-class/my-pod.yaml) 為例，

```
apiVersion: v1
kind: Pod
metadata:
  name: apiserver
  labels:
    app: apiserver
    tier: backend
spec:
  containers:
  - name: my-pod
    image: zxcvbnius/docker-demo
    ports:
    - containerPort: 3000
    volumeMounts:
    - name: my-pvc
      mountPath: "/tmp"
  volumes:
  - name: my-pvc
    persistentVolumeClaim:
      claimName: myclaim
```


可以看到在 [my-pod.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day21/demo-storage-class/my-pod.yaml) 中，多了兩個欄位 `spec.containers.volumeMounts` 與 `spec.volumes`，

- **spec.containers.volumeMounts.name**  
  我們指定的 volume 的名稱

- **spec.containers.volumeMounts.mountPath**  
  希望在 container 中掛載的路徑
  
- **spec.volumes.persistentVolumeClaim**  
  指定我們將使用的 PersistentVolumeClaim 物件的名稱

- **spec.volumes.name**  
  給定該 PersistentVolumeClaim 物件對應的名稱，供`spec.containers.volumeMounts.name` 使用

如此，當下次我們在創建 `my-pod` 時，Kubernetes 會幫我們去尋找相對應的資源，並將 Volume 掛載在該 Pod 底下。當 `my-pod` 消失時，我們也可以將檔案、資料保存下來囉！

# 總結
雖然今天沒有透過 `kubectl create` 或 `kubectl get` 等指令帶著讀者實做。然而在明天的學習筆記中，我們將延續先前 [在 AWS 上架設的 Kubernetes Cluster](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day15)，介紹如何將過去兩天所學習到的物件，在 AWS 上架設 [Stateful](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day07) 的 Wordpress Application。

# Q&A
最後，歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
