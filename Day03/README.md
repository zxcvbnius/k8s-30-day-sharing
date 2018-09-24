# [Day 3] 打造你的Docker containers


# 前言
在前兩天[介紹完Kubernetes](https://ithelp.ithome.com.tw/articles/10192401)，並且在[本機端成功架設Kubernetes cluster](https://ithelp.ithome.com.tw/articles/10192490)之後，想必對Kubernetes有些基本了解。今天想介紹的是，如何把程式打包成docker image，建造一個屬於你自己的container，並在日後能跑在Kubernetes上。今天學習筆記的大綱如下：

 - 安裝 Docker Engine
 - 將程式包成 Docker Image
 - 在本機上運行 Containerized App


> 如果對containers或是Docker還不熟悉的讀者，不妨參考以下文章，也許可以幫助你更暸解兩者是什麼：  
>
> - [Docker —— 從入門到實踐](https://philipzheng.gitbooks.io/docker_practice/content/)  
> - [ITHome - Container技術三部曲](https://www.ithome.com.tw/article/91838)




# 安裝 Docker Engine
在開始打造docker container之前，我們必須先在本機上安裝`Docker Engine`。Docker Engine支援許多種平台，在[官網](https://docs.docker.com/engine/installation/)上也列出所有支援的平台，而讀者可以在上面選擇與自己本機端相符的套件安裝。


![docker-installation](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-installation.png?raw=true)


而在今天的學習筆記裡，會帶大家在MacOS與Ubuntu16.04上安裝Docker Engine。



## 安裝在 MacOS
將docker engine安裝在Mac OS的步驟很間單：  

1) 在[官網](https://docs.docker.com/engine/installation/)找到[Docker for Mac (macOS)](https://docs.docker.com/docker-for-mac/install/)，點擊之後會進入頁面

![docker-installation-for-macos](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-installation-for-macos.png?raw=true)

直接點擊，下載套件後，在status bar看到一隻小鯨魚的符號就代表裝好囉。

![docker-run-on-macos](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-run-on-macos.png?raw=true)



## 安裝在 Ubuntu16.04

```
$ sudo apt-get update && sudo apt-get install docker.io
```

docker套件約90MB左右。安裝完成後，使用`groups`指令查看`目前user是否有docker group權限`

```
$ groups
ubuntu
```

若還沒有，輸入以下指令(如果user不是ubuntu，記得需要改成目前登入的username)

```
$ sudo usermod -G docker -a ubuntu
```
執行行完上述指令之後，記得`重新登入`，再次查看

```
$ groups  
ubuntu docker
```

可以發現多了`docker` group，這時執行`docker version`查看目前版本

```
$ docker version
Client:
 Version:      1.13.1
 API version:  1.26
 Go version:   go1.6.2
 Git commit:   092cba3
 Built:        Thu Nov  2 20:40:23 2017
 OS/Arch:      linux/amd64

Server:
 Version:      1.13.1
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.6.2
 Git commit:   092cba3
 Built:        Thu Nov  2 20:40:23 2017
 OS/Arch:      linux/amd64
 Experimental: false
```



# 將程式包成Docker Image
接下來我們將以一個Nodejs App為例，將該程式包成Docker Image，以下所有的程式碼都能在[docker-demo](https://github.com/zxcvbnius/k8s-30-day-sharing/tree/master/Day03/docker-demo)中找到。在開始打包之前，我們先來了解一下docker-demo中的這三個檔案分別是用來做什麼。

## Dockerfile
在將程式dockerize時，都需要一個專屬於該程式的Dockerfile。

以 [docker-demo/Dockerfile](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-demo/Dockerfile) 為例

```
FROM node:6.2.2
WORKDIR /app
ADD . /app
RUN npm install
EXPOSE 300
CMD npm start
```

 - `FROM node:6.2.2`  
   這行會載入程式需要的執行環境，會根據不同的需求下載不同的映像檔，這裡是指node v6.2.2

 - `WORKDIR /app`   
   在這個docker中的Linux即將會建立一個目錄`/app`

 - `ADD . /app`  
   代表會將與Dockerfile同一層的所有檔案加到Linux的`/app`目錄底下

 - `RUN npm install`  
   運行`npm install`，npm install會下載nodejs相依的libraries

 - `EXPOSE 300`   
   是指container對外的埠號，再與外界溝通時使用

 - `CMD npm start`   
   最後透過`npm start`會運行Nodejs App


> 在這裡我們只會介紹dockerfile中我們所需的指令  
> 更多Dockerfile的細節可參考 [《Docker —— 從入門到實踐­》](https://philipzheng.gitbooks.io/docker_practice/content/dockerfile/instructions.html)


## Nodejs App
在 [docker-demo](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-demo) 中，我們還可以找到另外兩個檔案 [index.js](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-demo/index.js) 與 [package.json](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-demo/package.json)。

 - **index.js**：我們會藉由 [express](https://www.npmjs.com/package/express) 這個套件架設一個小型的Nodejs API server，並會跑在`port 3000`上，且有一個endpoint會回傳`Hello World!`字串。
 - **package.json**：則會記載這個程式所需的套件，以及讓程式運行的指令`npm start`。

> 礙於篇幅，在學習筆記中並不會一一講解Nodejs API server每一行是在做什麼。  
> 有興趣學習Nodejs的讀者可以參考，[Node.js Taiwan 社群協作中文電子書](https://dca.gitbooks.io/nodejs-tw-wiki-book/content/)


### index.js

```
var express = require('express');
var app = express();
app.get('/', function(req, res) {
  res.send('Hello World!');
});
var server = app.listen(3000, function() {
  var host = server.address().address;
  var port = server.address().port;

  console.log("Example app listening at 'http://%s:%s'", host, port);
})
```

### package.json

```
{
  "name": "myapp",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.16.2"
  }
}
```

在了解完 docker-demo 裡所有的程式碼之後，我們可以在本機端新增一個資料夾

```
$ mkdir docker-demo
```

並將 [docker-demo](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-demo) 中的檔案下載到本機端，就可以透過`ls`看到本機端的資料夾的檔案，接下來我們就會透過docker指令將這個檔案打包成docker image囉。

![docker-nodejs-dir](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-nodejs-dir.png?raw=true)


## Docker Build

進到本機端的`docker-demo`資料夾後，可以輸入以下指令

```
$ docker build .
```

![docker-build](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-build.png?raw=true)

等到看到 `Successfully built 59f3e3615488`，代表docker image已經建置好了。59f3e3615488 代表你的Docker Image ID，每次都由系統產生的，所以若讀者產生出來的Image ID與圖片中不一致，是正常的。(如果想看更多docker build時會產生的log，可以參考 [docker-build.log](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-build.log) )

接著我們可以輸入`docker image ls`的指令，根據Image ID，我們可以找到剛剛建立好的docker image

![docker-image-ls](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-image-ls.png?raw=true)




# 在本機上運行containerized app

再把包好Docker Image之後，我們可以先在本機上透過docker指令跑起我們的docker container。記住你剛剛的Docker Image ID，並輸入指令`docker run`

```
$ docker run -p 3000:3000 -it 59f3e3615488
```

![docker-run](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-run.png?raw=true)

看到這裡代表docker image已經跑起來了，可以到瀏覽器輸入 [http://127.0.0.1:3000/](http://127.0.0.1:3000/)

便可以看到我們剛剛架設好的Nodejs API server。

![docker-nodejs-demo](https://github.com/zxcvbnius/k8s-30-day-sharing/blob/master/Day03/docker-nodejs-demo.png?raw=true)




# 總結
雖然今天並沒有琢磨在Kubernetes上的介紹，但Kubernetes的每個元件設計與Containers也都息息相關。希望在每個人花時間看完這篇文章之後，都有獨立打造屬於自己Docker Image的能力。在明天將會介紹如何將Docker Image上傳到Docker Hub。而在之後的學習筆記，則會繼續探索Kubernetes。



# 參考連結
 - [Docker —— 從入門到實踐](https://philipzheng.gitbooks.io/docker_practice/content/)
 - [Node.js Taiwan 社群協作中文電子書](https://dca.gitbooks.io/nodejs-tw-wiki-book/content/)
