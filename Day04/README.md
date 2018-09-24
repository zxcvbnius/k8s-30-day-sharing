# [Day 4] 上傳Docker Image到Docker Hub


# 前言
在昨天介紹到[如何打造自己的Docker Container](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03)之後，想分享如何將本機端打造好的Docker Image傳到Docker Registry上。今天學習筆記可以分成幾個部分：

 - 介紹 什麼是 Docker Registry
 - 介紹 Docker Hub
 - 將本機端的Docker Image上傳到Docker Hub
 - 打造 Docker Image 的一些準則
 - 常用的 Docker Image




# Docker Run 的限制
在昨天介紹到`docker run`指令，就可以讓docker container在本機上跑起來。然而，這樣`需要在每台機器上手動部署`的方式，不太適合在實際產品環境中操作。所以我們需要像Kubernetes這樣管理containers的系統來幫我們部署、管理每個container的狀態。所以，需要有個地方能讓Kubernetes隨時存取這些Docker Image，也就是 [Docker Registry](https://docs.docker.com/registry/)




# 什麼是 Docker Registry
[Docker Registry](https://docs.docker.com/registry/) 就像一個倉庫，裡面存放著各式各樣的Docker Image。倉庫可以是公開的；也可以是私有的，只允許特定人員存取這些Image。Docker官方有提供一個 [Docker Hub Registry](https://hub.docker.com/) ，在上面可以找到需多開源套件官方提供的docker image，而接下來的操作也都會圍繞在Docker Hub上。



# Docker Hub
[Docker Hub](https://hub.docker.com/) 如同 [Github](https://github.com/)，在Github上有許多的專案，每個人都可以上傳自己的專案，也可以下載別人的專案，有時也可以在某些開源專案，提問或提交自己的程式碼。同時，[Github](https://github.com/) 也提供私有與公開的專案類型，如果專案不想被公開，則需要申請[付費帳號](https://github.com/pricing)。而[Docker Hub](https://hub.docker.com/)則提供免費帳號一個私有repository的quota。


## Docker Login
若還沒有Docker Hub的讀者，可以先到[官網](https://hub.docker.com/)申請帳號。

![docker-hub-signup](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-hub-signup.png?raw=true)

在登入之後，可以看到如下圖的歡迎畫面

![docker-hub-welcome-page](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-hub-welcome-page.png?raw=true)

接著回到terminal上，輸入`docker login`指令，登入剛剛申請的帳號。由於Docker Hub是官方提供，若是docker沒有特別指定其他的Docker Registry，登入之後的Registry會直接連結到Docker Hub。

![docker-loing](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-loing.png?raw=true)

看到`Login Succeeded`代表登入成功。


如果是使用MacOS的讀者，也可以在狀態列右上角點選小鯨魚，從這裡登入。

![docker-login-for-mac](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-login-for-mac.png?raw=true)

登入成功後，我們就可以準備把Docker Image上傳到Docker Hub上了。




# 建立Docker Repository
在上傳Docker Image之前，我們必須先建立一個Docker Repository。登入之後在Docker Hub首頁右上角可以找到 **Create**，點選 **Create Repository**

![docker-hub-create-repo](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-hub-create-repo.png?raw=true)


可以輸入你想要的repository名稱，每個repository的前綴字都會是登入帳號，像我就是**zxcvbnius/docker-demo**。專案可以選擇公開或是不公開，最後按下**create按鈕**就可以提交。

![docker-hub-create-repo-1](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-hub-create-repo-1.png?raw=true)

提交成功之後會看到關於這個repository的基本資訊，等等我們上傳的Docker Image也可以在這個頁面上看到。

![docker-repo-info](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-repo-info.png?raw=true)


# 上傳Docker Image
在terminal輸入`docker image ls`找到昨天建立的docker Image，會發現我們還沒有給這個image給一個tag，如同底下的redis, mysql image。

![docker-image-pre-tag](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-image-pre-tag.png?raw=true)

我們可以使用`docker tag`指令，將這個docker image，標上tag，而這個tag就是我們剛剛建立的repository，指令如下，

```
$ docker tag 59f3e3615488 zxcvbnius/docker-demo
```

`tag`指令後面接的是Image ID，記得替換成自己的Image ID；repository前面也記得加上登入的帳號。

如此在重新輸入一次`docker image ls`，會看到之前打包好的image，現在也有repository name與TAG了。

![docker-image-tag](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-image-tag.png?raw=true)

由於我們沒有指定TAG是哪個版本，所以會是`latest`表示最新版本。可以在repository加入版本，如此，打包出來的docker image，後面的TAG就會從`latest`變成你所`指定的版本號`了，可以試試以下指令。

```
$ docker tag {DOCKER_IMAGE_ID} {YOUR_ACCOUNT_NAME}/{REPOSITORY_NAME}:1.0.0
```

最後，使用`docker push`指令，就可以將tag好的Image，上傳到指定的repository囉。

```
$ docker push zxcvbnius/docker-demo
```

![docker-push](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-push.png?raw=true)

最後再回到Docker Hub頁面上，點選`Tags`，就可以找到剛剛push上去的docker image。

![docker-image-on-docker-hub](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-image-on-docker-hub.png?raw=true)


# Docker Remarks
 - **在一個container只運行一個 one process in one container**
   在產生Docker Image的時候應避免把太多服務包在一起，通常一個docker image都在幾百MB左右。
 - **container中的data不會保存下來 All data in the container is not preserved**
   當一個container停止運作時，存在該container的資料也會隨之消失。必須藉由第三方儲存服務，將內部的data保存下來，在日後學習筆記也會介紹到`Volumes`，Kubernetes會藉由Volumes Component將data保存下來。




# 常用的Docker Image
在Docker Hub的[Explore頁面](https://hub.docker.com/explore/)上可以看到目前受歡迎的docker images。


![docker-hub-explore](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day04/docker-hub-explore.png?raw=true)


以Nginx為例，若想下載最新版的Nginx Image，只需要在本機端輸入

```
$ docker pull nginx
```

再用`docker images`指令查看，就會發現本機端多了一個Nginx的image，有興趣的讀者不妨試試看。




# 總結

 - 希望看完今天的學習筆記之後，讀者現在也有能力能將本機端的docker image上傳到docker registry

 - Kubernetest與Docker Registry是一個密不可分的關係，Kubernetes所需要的docker images，都可以從我們指定的Registry下載。

 - 雖然對於Docker Hub沒有太多介紹，但Docker Hub上的資源非常豐富。許多常在用的工具都有提供官方的Image，以[Redis](https://hub.docker.com/_/redis/)為例，可以直接使用`pull`指令下載，無須在再自己打造一個image。


在對Docker Container有基本認識後，未來幾天的學習筆記中，會一ㄧ介紹Kubernetes各個功能，一起欣賞Kubernetes的強大之處。




# Q&A
依舊歡迎大家給予建議與討論，如果能按個讚給些鼓勵也是很開心唷 ^_^




# 參考連結

 - [Github](https://github.com/)
 - [Docker Hub](https://hub.docker.com/)
 - [Docker —— 從入門到實踐](https://philipzheng.gitbooks.io/docker_practice/content/)
