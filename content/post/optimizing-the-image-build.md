---
title: "Dockerfile 撰寫與 Image 最佳化心得"
date: 2021-03-19T08:00:00+08:00
modified: 2021-03-19T08:00:00+08:00
thumbnailImage: "/images/2021/03/20/docker_logo.jpg"
thumbnailImagePosition: left
categories:
- 容器化技術
tags:
- Optimize
- Dockerfile
- Image
keywords:
- Dockerfile
- Image
- 最佳化
showPagination: true
draft: false
---

最近在產學合作的專案當中實作許多 Infra<sup>[1]</sup> 相關的佈建，尤其在 Dockerfile 的撰寫與 Image 的最小化上有一些小成果，因此補充閱讀ㄧ些資訊並結合一些經驗來分享順便以文防老。

<!--more-->

---

在 Docker development best practices<sup>[Ref 3]</sup> 中有提到四個要點，分別是：

1. 知道如何保持 Image 最小化
2. 了解在何處與如何儲存應用程式資料
3. 使用 CI/CD 來測試與部署
4. 區分開發與正式環境的實務方式

這篇主要是針對第一點來分享，並透過 Image 最小化來降低儲存空間、減少流量花費同時加快其載入速度，像在專案中客戶是使用 IBM Cloud 的虛擬主機服務，流量每一分一毫都是要付費的，因此協助他們減少開支也是我們開發者的任務之一，同時也能夠加快我們 Deploy 新版本的效率。

其實看完資料整理的時候才發現 Image 最佳化這件事情其實學問不深卻非常實用，整理同時也檢討自己專案並發現有許多可以改善的地方，可以說是一舉兩得，而基本上由於 Docker 本身也是遵循 Open Container Initiative (OCI)<sup>[2]</sup> 因此這篇文章也通用於 Podman 或 Buildah 等遵循 OCI 的開源專案。

---

- <sup>[1]</sup> Infra：為基礎設施 Infrastructure 的簡稱，在資訊相關單位通常是指協助開發、營運行為及產品服務的基礎系統及硬體
- <sup>[2]</sup> Open Container Initiative (OCI)：開放的容器管理架構，目的是為了使不同容器應用系統開發商能夠有一致的介面，促進容器化的流通使用

---

## Dockerfile

Dockerfile 是一個以文字描述如何自動化建構 Image 的檔案，通常在開源專案中很常看見使用來協助使用者快速建構適合執行該專案應用程式的 Image，裡面會描述從哪一個基本的 Image 開始建構、中間需要複製什麼檔案執行那些指令...直到最後應用程式啟動的流程，如果使用者不想使用容器化的佈署方式，透過參考 Dockerfile 也能夠了解其應用程式要如何安裝並且啟動。

### 指令介紹
> 主要參考 [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)，詳細撰寫方式可以參考這邊

##### FROM
簡單來說就是挑選你要以哪個 Image 為基礎向上延伸，例如開發 Java 應用程式可能就會使用 [OpenJDK](https://hub.docker.com/_/openjdk) 的 Image 來進行建構，如此一來描述就會像是 `FROM openjdk:8`，當然也可以使用自己產生的 Image 來延伸。

##### LABEL
用來針對建構好的 Image 進行標籤，例如：`LABEL module.auth.version="0.0.1"`，複數個標籤可以用單行空白隔開標籤 `LABEL A=1 B=2`，或一行一行宣告的方式來標籤，透過標籤的方式能夠更快速透過 `--filter` 來找到對應的單一或群組的 Image，當然標籤功能不僅限於 Image，在其他像是 Container、Network 或 Volume 上都有相同的功能可以產生 [Object labels](https://docs.docker.com/config/labels-custom-metadata/) 來方便尋找及管理。

##### RUN
如同在 Shell 中下指令般，用於建構 Image 的環境， `RUN` 可以協助執行複雜的指令並且能夠透過反斜線 `\` 來將指令分割成多行，提高可讀、可維護性，例如像 Docker 官方文件中提供的範例：

{{< codeblock "RUN example" >}}RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
&& rm -rf /var/lib/apt/lists/*{{< /codeblock >}}

這邊要特別注意的是 `RUN` 並不會拿來執行最後想要啟動的應用程式，如果要啟動應用程式則需要使用後面提到的 `CMD` 或 `ENTRYPOINT`。

{{< alert info >}}
Debian Ubuntu 官方 image 會自動執行 `apt clean`
{{< /alert >}}
{{< alert warning >}}
Docker Image 建構時預設使用 /bin/sh -c 執行，因此只要最後一個成功就會當作成功，所以 Pipe 失敗的行為可以用 `set -o pipefail &&` 來檢查，舉例來說：
```
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```
而有些 Image 可能預設的 Shell 不支援 -o pipefail 因此建議使用
```
RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
```
{{< /alert >}}

##### CMD
主要用於運行應用程式並且允許附帶任意參數，與稍後提到的 `ENTRYPOINT` 非常類似，但 `CMD` 可以理解成「預設的啟動指令」，基本上用法就是 `CMD ["應用程式", "參數一", "參數二"…]`，像是以 Java 執行 Jar file 為例就會是：`CMD ["java", "-jar", "app.jar"]` 來表示 `java -jar app.jar` 這個指令。


##### EXPOSE
用來指定這個 Image 產生的 Container 允許其他 Container 以哪一個 Port 來連接，如果要與 Host 端連接則需要用 `-p` 的方式來指定，例如像： `-p 8080:80` 就是將外面 Host 端的 8080 Port 導到 Container 的 80 Port 上，而 Expose 則是只有事先指定其他 Container 能以哪個 Port 來存取而已。

##### ENV
基本上就跟平常在 Shell 中宣告環境變數一樣，如果不想讓環境變數殘留在建構好的 Image 中（包含建構中的任何一層）則官方建議的寫法會如下：

{{< codeblock "Example for remove env variable" >}}FROM alpine
RUN export VERSION="1.0.0" \
    && echo $VERSION > ./version \
    && unset VERSION
CMD sh{{< /codeblock >}}

透過這樣的方式來確保最後產生的 Image 是不會具有 VERSION 的環境變數，但是如果將 `ENV` 與 `unset` 分成兩個步驟來寫，則 ENV 那一層會保留著環境變數，直到 unset 層才會被移除。

##### ADD / COPY
這兩個標籤功能極為類似，主要都是將檔案從本機或先前建構的環境中複製到當前執行的環境底下，許多開源專案一不小心還會交錯使用，但文件裡寫道：「一般來說 `COPY` 為首選」，而 `ADD` 有什麼特別的呢？基本上可以參考[文件](https://docs.docker.com/engine/reference/builder/#add)裡面也有詳細描述 ADD 所遵循的規則，主要是複製對象如果是常見的壓縮格式 (identity, gzip, bzip2 或 xz) 則會將其解壓縮成目錄，也可以於來源指定遠端的 URL，但若 URL 下載下來為常見壓縮格式則不會主動解壓縮。

除了從本機將檔案複製到當前環境底下之外，也可以從先前建構的環境底下複製，像是下面以 Java 應用程式打包為例，透過標記先前的建構 `gradle:6.4.0-jdk8` 取名為 `build` 後再當前環境將前一個環境中編譯好的 jar file 複製進來。

{{< codeblock "Dockerfile / Build for Java application" >}}FROM gradle:6.4.0-jdk8 AS build
COPY --chown=gradle:gradle . /home/gradle/src
WORKDIR /home/gradle/src
RUN gradle build

FROM openjdk:8-jre-slim
EXPOSE 4300
RUN mkdir /app

COPY --from=build /home/gradle/src/build/libs/*.jar /app/myService.jar{{< /codeblock >}}

##### ENTRYPOINT
設定 Image 中主要應用程式的進入點，通常會將固定不太會變動的部分放在 `ENTRYPOINT` 中，而會動態變化的則會建議以 `CMD` 附加在其後，可以參考下面藍色提示框中更詳細的說明。

##### VOLUME
指定需要針對哪個目錄進行掛載動作，會產生一個 Docker volume 來對指定的目錄進行掛載，如此一來資料就能被保存於 Docker volumn 之中

##### USER
用來指定要執行後續動作的使用者與群組。

##### WORKDIR
可以視為切換當前目錄，通常會寫絕對路徑來指定到特定目錄底下，相對目錄要特別注意，如果照下方的方式撰寫 (取自官方範例)：

{{< codeblock "Example for workdir" >}}WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
{{< /codeblock >}}

這樣你執行 pwd (取得當前目錄) 會是出現 `/a/b/c`

##### ONBUILD
這個指令很特別，是用於當其他人以你建構出來的這個 Image 為基礎去建構的時候才會執行 ONBUILD 後面指令，而 ONBUILD 後面是可以指定其他例如像是 `COPY` 或是 `RUN` 的指令。

##### ARG
用於建構時指定變數，使用時可以在 `docker build <path>` 後加上 `--build-arg` 帶入對應的參數與參數值。

##### HEALTHCHECK
使 Docker 以指定方法進行容器的健康(正常運作)檢查，例如像是 Web Server 就可以使用 `HEALTHCHECK --interval=5m --timeout=3s \ CMD curl -f http://localhost/ || exit 1` 的方式來確保是否有正常運作

##### SHELL
可以使用指定的 SHELL 來執行指令。

---

####  RUN vs CMD vs ENTRYPOINT 

{{< alert info >}}
針對比較容易搞混的 RUN / CMD / ENTRYPOINT 在這邊特別提出來做比較，當初自己在參考別人的 Dockerfile 時也常常看到各種方法混用及誤用，在這次順便整理成一個比較好理解的表格

|RUN|CMD|ENTRYPOINT|
|---|---|----------|
|用於建構環境|作為預設的啟動(參數)或互動指令，可以在啟動時取代|作為主要應用程式的進入點，無法再啟動時取代|

基於上述表格與官方文件提到的 Best pratice 以 Java 應用程式為例，則可以想像成 `ENTRYPOINT` 為 `java -jar myService.jar`，而 `CMD` 為 `-env=production`，而在這樣的編寫之下當啟動容器時可以透過指令的方式去覆寫後面的環境變數 `-env=production` 來使容器可以更為彈性的使用。
{{< /alert >}}

如果還是不太理解的話，這裡有一個小範例可以讓大家嘗試看看，先寫一個 dockerfile 在當前目錄，並且執行後面的指令來觀察看看結果：

{{< codeblock "Dockerfile" >}}FROM alpine:3.12.4
ENTRYPOINT ["ls"]
CMD ["-la", "/"]
{{< /codeblock >}}

{{< codeblock "bash / Add sudo if you need" >}}docker build --tag test/1 .
docker run test/1
docker run test/1 /
docker run test/1 -alh /
{{< /codeblock >}}

{{< image classes="fancybox fig-33" title="run test/1" src="/images/2021/03/20/result_entrypoint_cmd_1.jpg" thumbnail-height="200px" >}}
{{< image classes="fancybox fig-33" title="run test/1 /" src="/images/2021/03/20/result_entrypoint_cmd_2.jpg" thumbnail-height="200px" >}}
{{< image classes="fancybox fig-33 clear" title="run test/1 -alh /" src="/images/2021/03/20/result_entrypoint_cmd_3.jpg" thumbnail-height="200px" >}}

如此一來是不是就可以將 `ls` 作為主要程式的進入點，並且可以隨需求調整後面的參數了呢？兩者切割的好彈性沒煩擾 XD

---

## Image

大概理解 Dockerfile 的各項標籤執行的內容後，就要來看看 Image 是如何產生的？簡而言之就是使用 `docker build .` 之後，會依據 Dockerfile 中描述的各項指令去逐行執行，並且只要執行上述任意一個指令便會產生一個中間過渡層 "Layer"，這也就是為什麼常常聽到 Image 是一層一層疊上去的，可以想像每個 Layer 都是一個 Check point，這樣的機制可以使得相同的任務可以節省執行時間與運算成本，每個 Layer 產生後都會依據指令內容給予一個 ID，像是可以看到圖片中就產生了三個 Layer，而每一層都會有屬於自己的編號。

{{< image classes="center clear" title="執行結果 (使用前面的 ls -la / Dockerfile)" src="/images/2021/03/20/result_build.jpg" >}}

### 最佳化策略
> 參考 [Google - Best practices for building containers](https://cloud.google.com/solutions/best-practices-for-building-containers/#build-the-smallest-image-possible)

1. 確認環境需求並使用最符合環境的基礎 Image (Use the smallest base image possible)
2. 降低無意義的 Layer 層數 (Reduce the amount of clutter in your image)
3. 盡量使 Layer 可以共用 (Try to create images with common layers)

第一點主要是在於許多人會{{< hl-text danger >}}隨便找一個可用的 Image 以其作為基礎{{< /hl-text >}}或著是說{{< hl-text danger >}}將建構環境直接拿來執行應用程式{{< /hl-text >}}都是大忌，換句話說你只是要執行一個簡單的 1 + 1 = ? 的計算過程，何必需要使用工程計算機呢？

換個實際的例子，在專案中前端 (Angular) 曾經拿 Node.js 的環境執行 `npm install` 然後直接 `ng serve` 以測試的內建 Web Server 來打包這個前端環境，因此我們得到了一個肥滋滋的 1.X GB Image，但是如果將 `ng build` (也就是將 Angular 的程式全部打包成 html, css 與 js) 後，以大家熟悉的 Nginx Image(FROM nginx:alpine) 來打包的話，就能獲得一個 35 MB 不到的 Image，是不是很驚人？如果沒有特別需求，以更精簡的 Web Server 打包應該能夠再壓更低呢～

其中二跟三是需要取捨的，有時候為了能讓 Layer 能夠共用則必須把指令切割細一點，這部分需要考驗大家切的經驗與能力了，主要可以跟大家分享的建議是指令可以一次性使用 RUN 來完成，不必要產生過多的 Layer，畢竟每個 Layer 執行都會花額外的ㄧ些時間，可以看看參考文章裡的比較表：

{{< image classes="center clear" title="參考文章中的比較圖" src="/images/2021/03/20/comparison_from_google.jpg" >}}

### Lab - 實測 Cache 機制

為了能夠更了解文件中提到的指令對應的 Cache 行為，因此透過一個小實驗來加深印象，主要流程是會透過兩個檔案來確定 Cache 的 Layer 影響範圍

#### 環境

- Ubuntu 18.04.5 LTS
- Client: Docker Engine - Community
    > Version:           20.10.2

    > API version:       1.40
- Server: Docker Engine - Community
    > Version:          19.03.14
    
    > API version:      1.40 (minimum version 1.12)

#### 實作 (Lab)

首先我們會準備下面的 Dockerfile 與一個檔案 "A" 裡面寫 123，然後進行建構，如果自己嘗試時遇到 Permission Denied 可以自行在所有 docker 指令前面加上 sudo 哦！

1. 第一次建構 `docker build . --tag test`
> `--tag test` 指的是要將建構好的 image 標記成 test:latest 方便未來使用

> 如果想執行看看可以使用指令：`docker run test`

{{< codeblock "Dockerfile" >}}FROM ubuntu:focal
RUN apt-get update && apt-get install -y \
    vim \
    && rm -rf /var/lib/apt/lists/*

COPY ./A /

USER root

ENTRYPOINT ["cat", "/A"]
{{< /codeblock >}}

2. 更改 A 檔案後重新建構 `docker build . --tag test`

{{< image classes="fancybox fig-50" title="圖一、第一次建構 (前半部)" src="/images/2021/03/20/first_run_1.jpg" thumbnail-height="100%" >}}
{{< image classes="fancybox fig-50" title="圖二、第一次建構 (後半部)" src="/images/2021/03/20/first_run_2.jpg" thumbnail-height="100%" >}}
{{< image classes="fancybox fig-100" title="圖三、再跑一次第一次建構" src="/images/2021/03/20/first_run_repeat.jpg" >}}
{{< image classes="fancybox fig-100 clear" title="圖四、更改 A 檔案後重新建構" src="/images/2021/03/20/second_run.jpg" >}}

透過上面小小的 Lab 可以發現第一次建構時會逐行指令進行操作 (如上圖一、圖二)，建構完畢後重新建構一次會發現它都有找到對應的 Layer cache 因此就直接使用他們 (如上圖三)，接著修改檔案 A 之後，會看到 `Step 3` 之後都重新執行了 (如上圖四)，原因是因為 A 檔案有做過修改，因此透過 Docker 雜湊校驗的機制發現有所更動，因此就重新建構這一步往後的所有指令了，而這邊也能看到其實 `RUN` 的部分只要指令沒有更動，是不會主動去更新的哦！

{{< alert info >}}
如果想看一個 image 建構過程產生的所有 Layer 可以使用指令：
```
docker history <image name>
```
來查看建構的歷程，以我們 Lab 舉例的話就是使用 
```
docker history test
```
{{< /alert >}}

## 經驗提醒
> 這邊基本上是想到或遇到就來補充 XD

1. 像是常見的 `RUN apt update` 這類的指令並不是每次 Build 就會幫你重新下載更新，而是只要指令沒有變動則有舊的 Cache 就使用舊的，所以對此如果想要進行更新的話通常會在 `build` 後面加上參數 `--no-cache`，來讓該次的建構不使用 Cache Layer。
2. 建構環境與執行環境可以分開，不一定要在建構環境中直接執行應用程式，同時也可以降低 Image 大小
3. 注意過程中有無暫存檔案，像是 `apt update` 後就會產生一些暫存資料，而這些在 Image 當中也只是佔空間而已，因此通常都會在額外加上 `apt-get -y clean && rm -rf /var/lib/apt/lists/*` 這類的清除暫存操作 (官方的 Debian 與 Ubuntu Image 會自動執行 apt clean)

## Reference

- <sup>[Ref 1]</sup> https://cloud.google.com/solutions/best-practices-for-building-containers/
- <sup>[Ref 2]</sup> https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- <sup>[Ref 3]</sup> https://docs.docker.com/develop/dev-best-practices/

## 其他資源

- [OCI - Image Spec](https://github.com/opencontainers/image-spec)
- [Docker Expose Port](https://www.whitesourcesoftware.com/free-developer-tools/blog/docker-expose-port/)
- [Dockerfile best-practices for writing production-worthy Docker images](https://github.com/hexops/dockerfile)
- [10 things to avoid in docker containers](https://developers.redhat.com/blog/2016/02/24/10-things-to-avoid-in-docker-containers/)