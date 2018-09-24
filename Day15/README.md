# 前言
今天的學習筆記將分享幾個常用的好工具、以及介紹 [AWS](https://aws.amazon.com/tw/) 上幾個我們將會用到的功能，筆記內容如下：

- 介紹什麼是 [Kops](https://github.com/kubernetes/kops) 

- 在不同平台上安裝 [Kops](https://github.com/kubernetes/kops) 套件

- 申請 AWS 開發帳號 & 安裝 [awscli](https://aws.amazon.com/tw/cli/) 套件

- 在 [AWS Identity and Access Management](https://aws.amazon.com/tw/iam/) 新增一組管理帳號

- 在 [AWS S3](https://aws.amazon.com/tw/s3) 新增一個 bucket 供 Kubernetes 使用

- 透過 [AWS route53](https://aws.amazon.com/tw/route53/) 指定 Domain Name

# 什麼是 [Kops](https://github.com/kubernetes/kops) 
[Kops](https://github.com/kubernetes/kops) 是 Kubernetes 提供的一個套件。協助開發者處理 Kubernetes Cluster 在 [AWS](https://aws.amazon.com/tw) 或 [GCP](https://cloud.google.com/?hl=zh-tw) 的安裝與配置，讓我們可以在這些雲端服務上快速打造一個產品級別的 Kubernetes Cluster。


# 在不同平台上安裝 [Kops](https://github.com/kubernetes/kops) 套件
以下將分別介紹如何在 **MacOs、Linux**、以及 **Windows** 上安裝 Kops


## MacOS
Kops 安裝的方式也非常簡單，以 MacOS 的平台為例，可以透過 **brew** 安裝 Kops 套件，只需在終端機輸入以下指令，便可在 MacOS 本機端裝好 kops 套件。

```
$ brew update && brew install kops
```

安裝好之後輸入 `kops` 即可看到指令說明，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/kops-command.png?raw=true)

## Linux
以 Ubuntu 16.04 平台為例，從 Kops 官網提供的套件連結下載 Kops binary 檔，

```
$ curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
```

下載下來的 binary file 約為 `75MB`，接著 `新增執行權限`，

```
$ chmod +x kops-linux-amd64
```

最後將這個 binary file 移到， `/usr/local/bin` 底下：

```
$ sudo mv kops-linux-amd64 /usr/local/bin/kops
```

完成之後，在終端機輸入 

```
$ kops
```

就會出現與上圖相同的指令說明囉！

## Windows
目前 `Kops` 只支援 **Linux** 與 **MacOS** 平台，使用 **Windows** 的讀者，必須在本機上的跑起一個 Linux VM ，安裝 Kops 套件。雖然在今天的學習筆記不會詳細介紹 Windows 上的安裝步驟，但若讀者在安裝過程中有疑慮，也非常歡迎來信討論 : )

# 申請 AWS 開發帳號

若是還沒有 AWS 開發帳號的讀者，可至官網申請一組帳號。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-apply-account-page.png?raw=true)

第一年申請的帳號提供許多優惠，包含每個月 750 個小時的 t2.micro 使用量、5 GB 的 S3 儲存空間。對於想在 AWS 試跑 Kubernetes 的我們已非常足夠。更多提供的優惠可參考 [AWS Free Tier](https://aws.amazon.com/free/)。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-free-tier-page.png?raw=true)


> 更多申請帳號的資訊可參考 [oranwind - 在 AWS 上申請帳號](https://oranwind.org/-aws-zhu-ce-aws-zhang-hao/)

# 安裝 [awscli](https://aws.amazon.com/tw/cli/) 套件
申請完帳號後，我們需要在本機端安裝 [awscli](https://aws.amazon.com/tw/cli/) 套件：

若是使用 **MacOS** 的讀者，可在終端機輸入以下指令，

```
$ sudo easy_install pip && pip install awscli
```

若是使用 **Linux** 的讀者，則可輸入

```
$  sudo apt install awscli
```

安裝完之後，可以用 `aws --version` 指令確認目前版本，

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/awscli-version.png?raw=true)

# 在 [AWS IAM](https://aws.amazon.com/tw/iam/) 新增一組管理帳號
在安裝好 `awscli` 以及 `kops` 套件後，我們需要在 AWS IAM 中新增一組管理帳號供 `kops` 使用。

首先，登入 [AWS console](https://console.aws.amazon.com/)

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-console-page.png?raw=true)

進入到 console 頁面後，在搜尋 bar 底下輸入 `IAM` ，相關的選項便會顯示在螢幕上。點選該選項，會進到 IAM 設定頁面。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-search-iam-page.png?raw=true)

點選左邊的 **Users** 選項，會看到一個 **Add user** 的選項，點選進去後便可開始設定

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-iam-setting-page.png?raw=true)

接著輸入你想要的`使用者名稱`，這邊我使用 **kops**。由於我們稍後都會透過 `aws-cli` 來存取 AWS 上的服務，所以選擇 Programmatic access，設定完畢之後點選 **Next** 進到下一頁。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-iam-setting-page-2.png?raw=true)

接著，選擇 **"Attach existing policies directly"** ，並勾選 **AdministratorAccess**，讓 `kops` 這個 user 可以存取 AWS 上所有資源。確認無誤之後，點選 **Next: Review**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-iam-setting-page-3.png?raw=true)

再次確認無誤之後，按 **Create user**。如此就創建成功囉。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-iam-setting-page-4.png?raw=true)

最後，記下新帳號相對應的 **Access key ID** 與 **Secret access key**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-iam-setting-page-5.png?raw=true)

回到 terminal 上，輸入 `aws configure`，並打入剛剛我們記下的 **Access key ID** 與 **Secret access key**，最後兩個選項若沒特別指定按 `enter 鍵` 即可。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-configure.png?raw=true)

完成後，可以在 `$HOME/.aws/` 資料夾裡面，找到我們剛剛設定的值 ，

```
$ cat ~/.aws/credentials
[default]
aws_access_key_id = ...
aws_secret_access_key = ....
```

# 在 [AWS S3](https://aws.amazon.com/tw/s3) 新增一個 bucket 供 Kubernetes 使用

完成 IAM 的設定之後，我們將在 [AWS S3](https://aws.amazon.com/tw/s3) 新增一個 `bucket`，這個 `bucket` 裡面將會存放所有與 Kubernetes 相關的檔案。先找到左上角的 **Services** 選項，點選之後會看到 Storage 中有個 **s3**，點擊進去便會進入 s3 的設定頁面。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-search-s3.png?raw=true)

接著點選 **Create bucket**，會跳出一個對話視窗。輸入你想要的 bucket name，這邊我命名為 `k8s-demo-qwer`，另外也可以選擇這個 bucket 想放在哪個地區(Region)，讀者可以選擇與自己所在地相進的 Region。

> 值得注意的是，bucket name 必須是獨特的。若是與其他 bucket 撞名，AWS 會要求開發者使用其他名稱。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-s3-setting-page.png?raw=true&version=2)

完成設定之後，點選 **Next**。接著，必須給予我們剛剛設置好的 `kops` 帳號這個 bucket 的管理權限。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-s3-setting-page-2.png?raw=true&version=2)

先回到 terminal ，輸入 `aws iam` 指令，便可以找到該 user 的 `UserId`，並將這個 `UserId` 填入 s3 的設定頁面，同時選取所有 `Read`，`Write` 的選項。
完成之後點選 **Next**。

```
$ aws iam get-user --user-name kops
{
    "User": {
        "Path": "...",
        "UserName": "kops",
        "UserId": "...",
        "CreateDate": "...",
        "Arn": "..."
    }
} 
```

最後按下 **Create bucket** 就創建好一個 bucket 物件了！回到頁面上，也可以看到過剛創建好的 `k8s-demo-qwer` 的資訊。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-s3-setting-page-3.png?raw=tru)

# 透過 [AWS route53](https://aws.amazon.com/tw/route53/) 指定 Domain Name
最後，我們需要設置一個 Domain Name，讓外部服務都能從這個 Domain Name 訪問我們的服務。一樣在右上角點選 **Services**，搜尋 **Route 53**

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-search-route53-page.png?raw=true)

進到 Route 53 設定頁面後，有幾種選項。如果讀者本身已經有自己網域時，可以點選 **"DNS management"**，若是還沒有自己的網域的讀者，也可以點選 **"Domain registration"** 選項，從 AWS 上申請一組新的網域。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-route53-setting-page.png?raw=true)

由於筆者本身已在 [GoDaddy](https://tw.godaddy.com/) 擁有一組 [網域](http://zxcvbnius.com/)，所以選擇 **"DNS management"**

接著點選右上角的按鈕 **"create Hosted zone"** 後，會跑出右邊的設定欄。可以指定想要的 Domain Name。在示範中設置 `k8sdemo.zxcvbnius.com`。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-route53-setting-page-2.png?raw=true)

設定完之後點選 **儲存**，會看到 AWS 提供的NS設定值。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/aws-route53-setting-page-3.png?raw=true)

已筆者為例，筆者須將這些值放在 [GoDaddy](https://tw.godaddy.com/) 提供的管理介面底下。

![](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day15/godday-setting-record.png?raw=true)

如此，在 AWS 上架設 kubernetes 的前置作業都完成囉！

# 結論
今天並沒有對 AWS 的 [IAM](https://aws.amazon.com/iam/), [S3](https://aws.amazon.com/s3/), 以及 [Route 53](https://aws.amazon.com/route53/) 的功能多做介紹，對於不熟悉 AWS 的讀者可能稍嫌吃力。在 [參考連結]() 中，筆者也會陸續附上自己覺得不錯的文章，相信對想更深入了解這些功能的讀者會有所幫助。

# 參考連結
 - [AWS 官網文件](https://aws.amazon.com/tw)
 - [oranwind - 在 AWS 上申請帳號](https://oranwind.org/-aws-zhu-ce-aws-zhang-hao/)
 - [GoDaddy 網域註冊教學](https://maiyuguo.life/archives/2427)
 - [GoDaddy DNS 管理介紹](https://tw.godaddy.com/help/dns-680)

