# 前言
在某些時候我們會需要有個**週期性運行的服務**，像是每天早上出昨天網站流量的報表，或是每隔一段時間去訪問某個 API 取得目前該應用服務的最新資訊。像這樣每小時、每日、每週、每月每隔一段固定時間要做的工作，我們會丟到 Linux 的 [crontab](http://linux.vbird.org/linux_basic/0430cron.php)，而 Kubernetes 也提供了[Cronjob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) 元件幫我們實現排程服務。今天的學習筆記如下，

- 介紹什麼是 [Cronjob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- 實作：在 [minikube](https://github.com/kubernetes/minikube) 上實現排程服務

# 什麼是 [Cronjob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

如果對 Linux 上的 [crontab](http://linux.vbird.org/linux_basic/0430cron.php) 熟悉的讀者，應該不陌生以下這張圖

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/crontab-time-format.png?raw=true)

(圖片擷取自 [cron format](http://www.nncron.ru/help/EN/working/cron-format.htm))

從左到右分別代表
- 分鐘(0-59)
- 小時(0-23)
- 每個月第幾天(1-31)
- 月份(1-12)
- 每個禮拜的第幾天(0-6)

以在 MacOS 上示範為例

## Crontab on MacOS
我們可以在終端機輸入以下指令去編輯排程工作

```
$ crontab -e
```

輸入以下文字

```
*/1 * * * * echo "Hi, current time is $(date)" >> ~/cronjob.log
```

代表每相隔一分鐘，會將 echo print 出來的資料倒到 **cronjob.log** 該檔案中。設定完成後，也可以用 `crontab -l` 查看目前有哪些 cronjob

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/crontab-list.png?raw=true)

接著，我們也可以到 **cronjob.log** 查看是否真的有 log 印出來

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/crontab-log.png?raw=true)

# 實作：在 [minikube](https://github.com/kubernetes/minikube) 上實現排程服務
而在 [Kubernetes](https://kubernetes.io) 上也能實現相同的排程服務，以 [my-cronjob.yaml](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/demo-cronjob/my-cronjob.yaml)

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: apline
            args:
            - /bin/sh
            - -c
            - echo "Hi, current time is $(date)"
          restartPolicy: OnFailure
```

我們想透過該設定檔創建一個名稱為 `hello` 的 cronjob，在每一分鐘的時候都能 Kubernetes 上運行 `echo "Hi, current time is $(date)"` 的指令 

創建的指令如下，

```
$ kubectl create -f ./my-cronjob.yaml
cronjob "hello" created
```

用 `kubectl get` 查看，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/kubectl-get-cronjob-2.png?raw=true)

會發現還沒開始執行，再過一陣子之後再查看一次

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/kubectl-get-cronjob-3.png?raw=true)

會發現該 cronjob 已執行，此外也可以透過另一個指令即時查看目前 job 運行的狀況

```
$ kubectl get jobs --watch
NAME               DESIRED   SUCCESSFUL   AGE
hello-1516166340   1         1            3m
hello-1516166400   1         1            2m
hello-1516166460   1         1            1m
hello-1516166520   1         1            7s
....
```


## 查看 Logs
如果想查看 crontab 是否有正常執行，也可以透過 `kubectl logs` 指令。首先我們可以先透過 `kubectl get` 找到相對應的 Pod ，指令如下

```
$ kubectl get pods -a --show-all=true -o wide --show-labels=true
```

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day23/kubectl-get-cronjob-4.png?raw=true)

接著印出屬於 `hello` Cronjob 底下的 Pod 的 log，以 `hello-1516166760-b86p6` 為例

```
$ kubectl logs hello-1516166640-b86p6
Hi, current time is Wed Jan 17 05:24:15 UTC 2018
```

當然我們也可以查看其他 Pod 的 log，看是否執行時間相隔一分鐘多

```
$ kubectl logs hello-1516166700-k59cn
Hi, current time is Wed Jan 17 05:25:15 UTC 2018

$ kubectl logs hello-1516166760-hbzg5
Hi, current time is Wed Jan 17 05:26:15 UTC 2018
```

# 結論
過去 crontab 都是綁定在某台 Linux 的某個使用者帳號底下，若是 crontab 分散在各台 Linux VM，管理上多少有些不方便。而 Kubernetes 不只提供開發者 [Cronjob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) 元件部署排程服務，同時，Kubernetes 也提供了一個很好的管理介面讓我們查看目前正在運行的排程服務有哪些。只需要透過以下指令

```
$ kubectl get cronjobs --show-all=true --all-namespaces=true
```

便能找到在 Kubernetes 上所有的 Cronjob 囉。


# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ：）

# 參考連結
- [Kubernetes 官網](https://kubernetes.io)

