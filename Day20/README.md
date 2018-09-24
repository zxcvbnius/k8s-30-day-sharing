# 前言
在前幾天的學習筆記中，我們使用到的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 物件都是 [stateless](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13)，代表著 **[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 物件中的 container 儲存的資料會隨著 container 的生命週期消失而消失，而無法被保存下來**。而今天的學習筆記中，將介紹 Kubernetes 的另一個套件 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)，讓我們可以輕鬆打造一個 [stateful](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day13) 的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)。讓`這個 Pod 中的 container 即便因為某些因素而 crash ，資料仍可完整的被保存下來，讓新產生的 container 能延續使用`。

今天的學習筆記如下：
- 介紹什麼是 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- 實作：在 AWS Kubernetes Cluster 上綁定 [AWS EBS](https://aws.amazon.com/tw/ebs/) Volumes


小提醒：今天的程式碼都可以在 [demo-volumes](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day20/demo-volumes) 中找到

# 什麼是 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
[Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) 可以當成是 Kubernetes Cluster 中專們用來`儲存資料的地方`。不但能將 container  的資料儲存下來，也可以透過`掛載(mounting)`的方式，供**許多個 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 同時存取**。而 Kubernetes 也提供非常多種的 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) 類型，在 [Kubernetes 官網](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)中，對每個類型也都有非常詳細的介紹，若有興趣的讀者不妨參考看看。

而今天的學習筆記中，將介紹四種常用的 Volume 類型：
- emptyDir
- hostPath
- Cloud Storage
- NFS (Network FileSystem)

## [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
每當我們建立一個新的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 物件時，Kubernetes 就會在這個 Pod 裏建立一個 [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)，`該 Pod 中所有的 container 都可以讀寫 emptyDir 中的資料`。當 Pod 從 Node 中被移除時，emptyDir 也會隨之消失，emptyDir 有以下幾個用途：

- **暫時性儲存空間**
  例如某些應用程式運行時需要一些臨時而無需永久保存的資料夾
- **共用儲存空間**  
  正如上述提到，同一個 Pod 中所有的 containers 都可以讀寫 emptyDir，也可以將 emptyDir 當作是這些 containers 的共用目錄

範例如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```


## [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 物件上，掛載 [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) 的資料夾或檔案。**hostPath 的生命週期與 Node 相同，當 Pod 因某些原因而須重啟時，檔案仍保存在 Node 的檔案系統(file system)底下，直到該 Node 物件被 Kubernetes Cluster 移除，資料才會消失**。

以 [hostpath-example.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day20/demo-volumes/hostpath-example.yaml?raw=true) 為例，

```
apiVersion: v1
kind: Pod
metadata:
  name: apiserver
spec:
  containers:
  - name: apiserver
    image: zxcvbnius/docker-demo
    volumeMounts:
    - mountPath: /tmp
      name: tmp-volume
    imagePullPolicy: Always
  volumes:
  - name: tmp-volume
    hostPath:
      path: /tmp
      type: Directory
```


在這個 [YAML](https://zh.wikipedia.org/zh-tw/YAML) 檔中，我們把 **Node 的 `/tmp` 掛在 apiserver 的 `/tmp` 下**。當 apiserver 的 **/tmp** 新增檔案時，可以從 Node 的 **/tmp** 中底下找到該檔案，因為兩者存取相同的資源。

用 `kubectl create` 創建 `apiserver` 物件，指令如下：

```
$ kubectl create -f ./hostpath-example.yaml
pod "apiserver" created
```

接著，進到 `apiserver` 的 shell 中，找到 `/tmp`，並新增一個 **test.txt** 的檔案，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day20/kubectl-exec-apiserver.png?raw=true) 

接著進到 `minikube` 的 shell 中，可以找到剛剛我們建立的 `test.txt` 檔案，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day20/minikube-cat-test-text.png?raw=true) 

若該 Pod 遭刪除，仍會保存在 `/tmp/test.txt` 中供新的 Pod 物件使用。

## Cloud Storage
此外，Kubernetes 也支援 [AWS EBS](https://aws.amazon.com/tw/ebs/)、[Google Disk](https://cloud.google.com/persistent-disk/?hl=zh-tw) 與 [Microsoft Azure Disk](https://azure.microsoft.com/en-us/services/storage/unmanaged-disks/) 等雲端硬碟類型的 Volumes。在今天的實作中，會介紹如何將 [AWS EBS](https://aws.amazon.com/tw/ebs/) 掛載在 [minikube](https://github.com/kubernetes/minikube) 的 Pod 上。

## [NFS (Network FileSystem)](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)
此外，Kubernetes Volumes 也支援 NFS，以 [nfs-example.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day20/demo-volumes/nfs-example.yaml?raw=true) 為例，

```
apiVersion: v1
kind: Pod
metadata:
  name: apiserver
spec:
  containers:
  - name: apiserver
    image: zxcvbnius/docker-demo
    ports:
      - name: api-port
        containerPort: 3000
    volumeMounts:
      - name: nfs-volumes
        mountPath: /tmp
  volumes:
  - name: nfs-volumes
    nfs:
     server: {YOUR_NFS_SERVER_URL}
     path: /
```

> 關於 Kubernetes 上更多 NFS 的範例，可參考[該連結](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs)

在介紹完幾項不同類型的 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) 後，今天的學習筆記將帶讀著在 [minikube](https://github.com/kubernetes/minikube) 上掛載 [Amazon Elastic Block Store](https://aws.amazon.com/tw/ebs/) 


# 實作：在 AWS Kubernetes Cluster 上綁定 [AWS EBS](https://aws.amazon.com/tw/ebs/) Volumes
> 若是還不熟悉怎麼在 AWS 上架設 Kubernetes 或還沒安裝 `aws-cli` 的讀者，不妨先閱讀 [[Day 15] 介紹 kops - 在 AWS 上打造 Kubernetes Cluster (上)](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day15)。

透過 `aws-cli` 我們在本機上創建一個架設在 [Oregon](https://docs.aws.amazon.com/general/latest/gr/rande.html) 且大小為 `1GB` 的 EBS, 指令如下：

```
$ aws ec2 create-volume \
> --size 1 \
> --region us-west-2 \
> --availability-zone us-west-2a \
> --volume-type gp2

{
    "VolumeId": "vol-0b29e0a08749ccef3",
    "SnapshotId": "",
    "Size": 1,
    "VolumeType": "gp2",
    "State": "creating",
    "Iops": 100,
    "CreateTime": "2018-01-08T04:38:58.885Z",
    "AvailabilityZone": "us-west-2a",
    "Encrypted": false
}
```

AWS  提供 5 種不同類型的 Volumes，若有興趣的讀者可以參考[該連結](https://aws.amazon.com/ebs/pricing/)，而在這次的實作中我們使用 `一般用途的 SSD (gp2)`。創建好後，我們可以在 [AWS console 的 EC2 頁面中](https://aws.amazon.com/tw/console/)看到我們剛剛建立好的 EBS，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day20/aws-ebs-detail.png?raw=true) 

或是，可以從 `aws ec2` 指令查詢新創建的 ebs 的詳細資料，指令如下，

```
$ aws ec2 describe-volumes --region us-west-2

{
    "Volumes": [
        {
            "Size": 1,
            "SnapshotId": "",
            "VolumeId": "vol-0b29e0a08749ccef3",
            "Attachments": [],
            "CreateTime": "2018-01-08T04:38:58.885Z",
            "AvailabilityZone": "us-west-2a",
            "VolumeType": "gp2",
            "State": "available",
            "Encrypted": false,
            "Iops": 100
        }
    ]
}
```

建立好 ebs 物件後，我們需要透過架設在 AWS 的 Kubernetes Cluster 中建立一個 pod，且這個 pod 掛載 ebs。以 [aws-ebs-example.yaml]() 內容為例：

```
apiVersion: v1
kind: Pod
metadata:
  name: apiserver
spec:
  containers:
  - name: apiserver
    image: zxcvbnius/docker-demo
    ports:
      - name: api-port
        containerPort: 3000
    volumeMounts:
      - name: aws-ebs-volumes
        mountPath: /tmp
  volumes:
  - name: aws-ebs-volumes
    awsElasticBlockStore:
     # replace to your volumeID
     volumeID: vol-0b29e0a08749ccef3
```

在這個設定檔中，我們希望這個 `vol-0b29e0a08749ccef3` 物件掛載在 `apiserver` 的 **/tmp** 資料夾底下。透過 `kubectl create` 創建，

```
$ kubectl create -f ./aws-ebs-example.yaml
pod "apiserver" created
```

成功之後，用 `kubectl describe` 查看，可以發現在 

```
$ kubectl describe pod apiserver

....
Volumes:
  aws-ebs-volumes:
    Type:  AWSElasticBlockStore 
    VolumeID:   vol-0b29e0a08749ccef3
```

Volumes 欄位底下看到我們掛載的 `vol-0b29e0a08749ccef3`。如此一來，當我們在 `apiserver` 的 **/tmp** 資料夾底下，新增或修改的檔案都會存放在 EBS 中。即便 `apiserver` 不存在了，當下次新的 Pod 一樣可以從 `vol-0b29e0a08749ccef3` 找到資料。

最後但也是最重要的是，**[AWS Elastic Block Store](https://aws.amazon.com/tw/ebs/) 使用上有一個很大的限制是：AWS EBS 只能被綁定在 EC2 中，也就是 Node 一定要架設在 AWS 上，才能綁定 EBS**。請讀者在使用 EBS 時務必留意囉。

# 總結
在最後的實作中，我們透過 `aws-cli` 的指令來產生一個 AWS EBS 物件。明天的學習筆記中將介紹， Kuberntes 提供了我們另一個方式，讓我們不再需要透過外部指令，可以從 `Kebuctl 指令 會 YAML 設定檔動態產生我們需要的 Volumes`。對運維以及管理多個 Volumes，也非常有幫助！

# Q&A
最後，歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)
- [Amazon Elastic Block Store](https://aws.amazon.com/tw/ebs/)
