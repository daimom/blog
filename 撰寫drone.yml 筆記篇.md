drone筆記篇
前言
  換工作後，開始接觸k8s、GCP、GKE，再來是Drone。
k8s中又延伸了kustomize ，ymal的寫法、佈署方式。
GCP中的cloud armor、GCE ，然後GKE的架構
真的是....滿滿的坑阿。以上講到的，還處於懞懂中，
筆記不知道何年何月何日可寫

正文
  簡單說一下環境，在GKE上面佈署 drone 跟 gitlab，
當gitlab有條件觸發的話(例如在push 在master上)，會同時送出一個webhook給drone，
此時drone就會去gitlab  pull程式碼下來，做你想要的動作（例如，打包成image，把image送到gcr上面，再佈署到GKE上）。

先來份yaml，建議可以先看一下yaml是什麼東西，這樣寫起來比較不會撞牆。
(ref. https://zh.wikipedia.org/wiki/YAML)


kind: pipeline
name: Pipeline(branch)

steps:      
- name: push2GCR(前哨)
  image: plugins/gcr

  settings:
    repo: gcr.io/rd7-project/landingpage
    tags:
      - latest
      - ${DRONE_COMMIT}
    # dockerfile: configs/beta/Dockerfile
    json_key:
      from_secret: GOOGLE_CREDENTIALS
    build_args:
      - website=beta

- name: deploy2GKE(前哨)
  image: nytimes/drone-gke
  environment:
    TOKEN:
      from_secret: GOOGLE_CREDENTIALS
  settings:
    project: rd7-project
    # 改拉參數傳入kube.yml內
    # template: configs/beta/.kube.yml
    vars:
      deployName: yaboxxx-landing-page-beta
      env: beta
    zone: asia-east1-a
    cluster: xbb-common
trigger:   
  # branch:
  #   - master
  ref:
    include:
      - refs/heads/master
  event:
    - push

分成三小塊來看，
如果以 --- 區分的話，那
kind: pipeline
name: Pipeline(branch)
是必須的。

再來是 steps ，
根據步驟游上往下執行，如果要并行的話，則使用depend_on ，這部分留到下次講。
1.  trigger ，這是觸發條件，表示在gitlab上面做了哪些動作，會觸發這個step（步驟）
  這邊的觸發事件，常用到應該是 Branch、Reference 以及 Event
  (ref. https://docs.drone.io/pipeline/docker/syntax/trigger/)
  Branch 跟 Event 比較好理解（但碰到Tag的事件又是另一回事了） 。
  Reference 這個其實只要在你的git裡面下指令
  tree .git/refs/
  大概就會知道他在講些什麼，git的底層都是靠refs去用出來的
  (
    ref.
    http://iissnan.com/progit/html/zh-tw/ch9_3.html
    https://titangene.github.io/article/git-branch-ref.html
  )
