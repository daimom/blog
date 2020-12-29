k8s yaml撰寫 volume 踩坑篇
前言
昨天接到一個任務，要把一個純html，放到nginx上面。
然後就開始踩坑之旅了
正文

這邊寫的都是GKE 的佈署方式，
關於GKE  文章實在太多了，我也不知道何年何月何日才有那個技術存量可以把它寫成文章。

目前使用的是 kustomize的方式佈署網站，
目前也在架設drone，日後走的是自動佈署，
但是第一次佈署還是要自己來的。

基本的流程是，將網頁包成一個 docker image，
(不論是php 或是 html， Go 則是本身有服務可以掛載，比較簡單點。)
再將此image設定成 initContainers ，
將此image的資料夾，掛載成volume，讓nginx可以直接掛載此資料夾。

這次踩的坑有兩個，
1. 網站程式，是別人傳給我的，所以檔案權限，everyone都是 無法存取。
所以要把裡面的所有檔案改成 755 rwxr-xr-x ，nginx圖片才能夠讀取。
我懶的一個一個改，所以下指令一次把整個資料夾內容都改掉，當初安全點是設定777
chmod -R 755 resource
(ref. https://www.jianshu.com/p/6c52260b1ff3)

2. 在寫yaml的時候，沒有搞清楚 volume的用途
導致一直出現rsync的錯誤。

initContainers:
- command:
  - sh
  - -c
  - |
    rsync -avrh --delete /source/* /app
  image: gcr.io/project/busybox-web:v2.7
  imagePullPolicy: Always
  name: source
  volumeMounts:
  - mountPath: /app
    name: source
volumes:
- emptyDir: {}
  name: source

volumeMounts: 這段的意思是 定義一個叫 soure的新磁區，掛載在 busybox-web容器裡的 /app 的path上面。
volumes: 的意思是定義磁碟區，emptyDir是伴隨著pod的生命，當pod消失資料也會跟著消失

3. nginx.conf 改完後，要重啟pod ，因為nginx.conf 是寫在 k8s的 configMapGenerator，
所以更改裡面的設定後，不會重啟pod ，裡面的服務就不會重新抓取conf。
