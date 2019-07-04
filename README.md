# Traefik

Traefik（發音為traffic），是一套用Go語言所開發的應用程式，2015年9月推出了v1.0，至今已經到了v1.7版本了。其主要功能是反向代理與負載平衡等功能Traefik，可以跟 Docker 很深度的結合，只要服務跑在 Docker 上面，Traefik 都可以自動偵測到。

反向代理 : 
在電腦網路中是代理伺服器的一種。伺服器根據用戶端的請求，從其關聯的一組或多組後端伺服器（如Web伺服器）上取得資源，然後再將這些資源返回給用戶端，用戶端只會得知反向代理的IP位址，而不知道在代理伺服器後面的伺服器叢集的存在
( example: www.example.com 指向 server 的 8080port)

負載平衡（Load balancing） : 
是一種電腦技術，用來在多個電腦（電腦叢集）、網路連接、CPU、磁碟驅動器或其他資源中分配負載，以達到最佳化資源使用、最大化吞吐率、最小化回應時間、同時避免過載的目的。

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


## Traefik反向代理

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

下面我們使用上述反向代理的方式把server上10080port(gitlab)只到domain name

一樣我們先撰寫gitlab的docker-compose.yml，主要是安裝redis,postgres還有gitlab，如下:

```
version: '3'

services:
  redis:
    restart: always
    image: redis
    command:
    - --loglevel warning
    volumes:
    - /srv/docker/gitlab/redis:/var/lib/redis:Z

  postgresql:
    restart: always
    image: postgres
    volumes:
    - /srv/docker/gitlab/postgresql:/var/lib/postgresql:Z
    environment:
    - DB_USER=gitlab
    - DB_PASS=password
    - DB_NAME=gitlabhq_production
    - DB_EXTENSION=pg_trgm

  gitlab:
    restart: always
    image: gitlab/gitlab-ce:11.10.4-ce.0
    depends_on:
    - redis
    - postgresql
    ports:
    - "10080:80"
    - "10022:22"
    volumes:
    - /srv/docker/gitlab/gitlab:/home/git/data:Z
    environment:
    - DEBUG=false

    - DB_ADAPTER=postgresql
    - DB_HOST=postgresql
    - DB_PORT=5432
    - DB_USER=gitlab
    - DB_PASS=password
    - DB_NAME=gitlabhq_production

    - REDIS_HOST=redis
    - REDIS_PORT=6379

    - TZ=Asia/Kolkata
    - GITLAB_TIMEZONE=Kolkata

    - GITLAB_HTTPS=false
    - SSL_SELF_SIGNED=false

    - GITLAB_HOST=localhost
    - GITLAB_PORT=10080
    - GITLAB_SSH_PORT=10022
    - GITLAB_RELATIVE_URL_ROOT=
    - GITLAB_SECRETS_DB_KEY_BASE=long-and-random-alphanumeric-string
    - GITLAB_SECRETS_SECRET_KEY_BASE=long-and-random-alphanumeric-string
    - GITLAB_SECRETS_OTP_KEY_BASE=long-and-random-alphanumeric-string

    - GITLAB_ROOT_PASSWORD=
    - GITLAB_ROOT_EMAIL=

    - GITLAB_NOTIFY_ON_BROKEN_BUILDS=true
    - GITLAB_NOTIFY_PUSHER=false

    - GITLAB_EMAIL=notifications@example.com
    - GITLAB_EMAIL_REPLY_TO=noreply@example.com
    - GITLAB_INCOMING_EMAIL_ADDRESS=reply@example.com

    - GITLAB_BACKUP_SCHEDULE=daily
    - GITLAB_BACKUP_TIME=01:00

    - SMTP_ENABLED=false
    - SMTP_DOMAIN=www.example.com
    - SMTP_HOST=smtp.gmail.com
    - SMTP_PORT=587
    - SMTP_USER=mailer@example.com
    - SMTP_PASS=password
    - SMTP_STARTTLS=true
    - SMTP_AUTHENTICATION=login

    - IMAP_ENABLED=false
    - IMAP_HOST=imap.gmail.com
    - IMAP_PORT=993
    - IMAP_USER=mailer@example.com
    - IMAP_PASS=password
    - IMAP_SSL=true
    - IMAP_STARTTLS=false

    - OAUTH_ENABLED=false
    - OAUTH_AUTO_SIGN_IN_WITH_PROVIDER=
    - OAUTH_ALLOW_SSO=
    - OAUTH_BLOCK_AUTO_CREATED_USERS=true
    - OAUTH_AUTO_LINK_LDAP_USER=false
    - OAUTH_AUTO_LINK_SAML_USER=false
    - OAUTH_EXTERNAL_PROVIDERS=

    - OAUTH_CAS3_LABEL=cas3
    - OAUTH_CAS3_SERVER=
    - OAUTH_CAS3_DISABLE_SSL_VERIFICATION=false
    - OAUTH_CAS3_LOGIN_URL=/cas/login
    - OAUTH_CAS3_VALIDATE_URL=/cas/p3/serviceValidate
    - OAUTH_CAS3_LOGOUT_URL=/cas/logout

    - OAUTH_GOOGLE_API_KEY=
    - OAUTH_GOOGLE_APP_SECRET=
    - OAUTH_GOOGLE_RESTRICT_DOMAIN=

    - OAUTH_FACEBOOK_API_KEY=
    - OAUTH_FACEBOOK_APP_SECRET=

    - OAUTH_TWITTER_API_KEY=
    - OAUTH_TWITTER_APP_SECRET=

    - OAUTH_GITHUB_API_KEY=
    - OAUTH_GITHUB_APP_SECRET=
    - OAUTH_GITHUB_URL=
    - OAUTH_GITHUB_VERIFY_SSL=

    - OAUTH_GITLAB_API_KEY=
    - OAUTH_GITLAB_APP_SECRET=

    - OAUTH_BITBUCKET_API_KEY=
    - OAUTH_BITBUCKET_APP_SECRET=

    - OAUTH_SAML_ASSERTION_CONSUMER_SERVICE_URL=
    - OAUTH_SAML_IDP_CERT_FINGERPRINT=
    - OAUTH_SAML_IDP_SSO_TARGET_URL=
    - OAUTH_SAML_ISSUER=
    - OAUTH_SAML_LABEL="Our SAML Provider"
    - OAUTH_SAML_NAME_IDENTIFIER_FORMAT=urn:oasis:names:tc:SAML:2.0:nameid-format:transient
    - OAUTH_SAML_GROUPS_ATTRIBUTE=
    - OAUTH_SAML_EXTERNAL_GROUPS=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_EMAIL=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_NAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_USERNAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_FIRST_NAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_LAST_NAME=

    - OAUTH_CROWD_SERVER_URL=
    - OAUTH_CROWD_APP_NAME=
    - OAUTH_CROWD_APP_PASSWORD=

    - OAUTH_AUTH0_CLIENT_ID=
    - OAUTH_AUTH0_CLIENT_SECRET=
    - OAUTH_AUTH0_DOMAIN=

    - OAUTH_AZURE_API_KEY=
    - OAUTH_AZURE_API_SECRET=
    - OAUTH_AZURE_TENANT_ID=
```
在最下方加入
```
    labels:
    - traefik.docker.network=traefik-network
    - traefik.frontend.rule=Host:gitlab.9skin.com
    - traefik.enable=true
    - traefik.port=80
    - traefik.default.protocol=http
```
 - traefik.port=80 的80port，指的是和本機10080port相對應的80port(上述文件內的"10080:80")
 
 ![image](https://github.com/leoa12412a/Traefik/blob/master/1.PNG)

 ## Gitlab的備份與還原，從實際gitlab備份到docker建立的gitlab
還原gitlab需兩個gitlab版本一致
首先使用指令備份gitlab
```
gitlab-rake gitlab:backup:create
```
備份出來的資料會在/var/opt/gitlab/backups裡面，會是一個.tar的壓縮檔
上面gitlab的yml裡面的 /srv/docker/gitlab/gitlab:/home/git/data:Z 我們可以知道本機的/srv/docker/gitlab/gitlab和container內的/home/git/data是雙向的，所以我們把備份出來的資料放到/srv/docker/gitlab/gitlab
```
mv <YOURFILENAME>.tar /srv/docker/gitlab/gitlab
```
如果是不同機器也可以使用scp傳送
```
scp <YOURFILENAME>.tar root@XXX.XXX.XXX.XXX:/srv/docker/gitlab/gitlab
```
移動完成後，查看gitlab container的id 或是 name
```
[root@122-147-213-61 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                 PORTS                                                   NAMES
94f5b4f8e391        gitlab/gitlab-ce    "/assets/wrapper"        5 hours ago         Up 5 hours (healthy)   443/tcp, 0.0.0.0:10022->22/tcp, 0.0.0.0:10080->80/tcp   dockercompose_gitlab_1
0cd070261663        postgres            "docker-entrypoint.s潀"   5 hours ago         Up 5 hours             5432/tcp                                                dockercompose_postgresql_1
8ac129bdaf50        redis               "docker-entrypoint.s潀"   5 hours ago         Up 5 hours             6379/tcp                                                dockercompose_redis_1
2ad8df6c9088        traefik             "/traefik --api --do潀"   5 hours ago         Up 5 hours             0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp              dockercompose_reverse-proxy_1
```
使用指令進入container
```
docker exec -it 94f5b4f8e391 bash
```
進入後把/home/git/data下的tar檔移動到gitlab預設backups下(/var/opt/gitlab/backups)

斷開 gitlab 與資料庫的連接
```
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
```
查看狀態
```
gitlab-ctl status
```
還原備份以及重新建立資料庫
```
gitlab-rake gitlab:backup:restore BACKUP=1493107454_2017_04_25_9.1.0
```
重新啟動gitlab
```
gitlab-ctl restart
gitlab-rake gitlab:check SANITIZE=true
```
![image](https://github.com/leoa12412a/Traefik/blob/master/2.PNG)

註 : 由於備份時因版本不同，所以我升級舊版gitlab v11 => v12 ，匯出備份後產生錯誤(升級前並未產生)，當下並未釐清錯誤原因，是否因此導還原錯誤還有待證實(2019.07.03)
