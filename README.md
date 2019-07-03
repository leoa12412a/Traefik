# Traefik

Traefik（發音為traffic），是一套用Go語言所開發的應用程式，2015年9月推出了v1.0，至今已經到了v1.7版本了。其主要功能是反向代理與負載平衡等功能Traefik，可以跟 Docker 很深度的結合，只要服務跑在 Docker 上面，Traefik 都可以自動偵測到。

反向代理 : 在電腦網路中是代理伺服器的一種。伺服器根據用戶端的請求，從其關聯的一組或多組後端伺服器（如Web伺服器）上取得資源，然後再將這些資源返回給用戶端，用戶端只會得知反向代理的IP位址，而不知道在代理伺服器後面的伺服器叢集的存在
( example: www.example.com 指向 server 的 8080port)

負載平衡（Load balancing） : 是一種電腦技術，用來在多個電腦（電腦叢集）、網路連接、CPU、磁碟驅動器或其他資源中分配負載，以達到最佳化資源使用、最大化吞吐率、最小化回應時間、同時避免過載的目的。

## 安裝Traefik在centos上

根據Traefik官方文件，使用我們Docker compose來安裝，Docker compose的安裝方法可以看<a href="https://github.com/a121514191/docker_compose">這裡</a>

首先我們先在server找一個自己喜歡的地方建立一個docker-compose.yml的檔案(你也可以較其他名字，指令要改成 docker-compose -f <YOURFILENAME> up -d)
  
```
[root@122-147-213-61 traefik]# pwd
/home/docker-compose/traefik
[root@122-147-213-61 traefik]# ls
[root@122-147-213-61 traefik]# touch docker-compose.yml
[root@122-147-213-61 traefik]# ls
docker-compose.yml
```

在docker-compose.yml寫入官方的文件所提供的內容，只要就是設定port以及下載traefik的image以及使traefik可以監聽所有docker事件

```
version: '3'

services:
  reverse-proxy:
    image: traefik # The official Traefik docker image
    command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
      - "8080:8080" # The Web UI (enabled by --api)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
```

執行docker-compose裡的reverse-proxy，你可以使用docker-compose up -d 他就會執行整個yml黨，下面輸入 docker ps -a 可以看到traefik的Container已經架設起來了。

```
[root@122-147-213-61 traefik]# docker-compose up -d reverse-proxy
Creating traefik_reverse-proxy_1 ... done
[root@122-147-213-61 traefik]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                                  NAMES 
4d198a84af41        traefik                    "/traefik --api --do潀"   53 seconds ago      Up 52 seconds       0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp              traefik_reverse-proxy_1
```
架設起來後，可以在8080port上看到traefik的web ui

![image](https://github.com/leoa12412a/Traefik/blob/master/t_dashboard.PNG)

接下來在docker-compose.yml內加入一個whoami的image來測試反向代理
```
# ...
  whoami:
    image: containous/whoami # A container that exposes an API to show its IP address
    labels:
      - "traefik.frontend.rule=Host:gitlab.9skin.com"
```
執行docker-compose裡的whoami，執行成功就可以在gitlab.9skin.com裡看到whoami了。
```
[root@122-147-213-61 traefik]# docker-compose up -d whoami
Creating traefik_whoami_1 ... done
```

```
gitlab-rake gitlab:backup:create
```
