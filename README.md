# Traefik

Traefik（發音為traffic）

```
[root@122-147-213-61 traefik]# pwd
/home/docker-compose/traefik
[root@122-147-213-61 traefik]# ls
[root@122-147-213-61 traefik]# touch docker-compose.yml
[root@122-147-213-61 traefik]# ls
docker-compose.yml
```

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

```
[root@122-147-213-61 traefik]# docker-compose up -d reverse-proxy
Creating traefik_reverse-proxy_1 ... done
[root@122-147-213-61 traefik]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                                  NAMES 
4d198a84af41        traefik                    "/traefik --api --do潀"   53 seconds ago      Up 52 seconds       0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp              traefik_reverse-proxy_1
```

```
# ...
  whoami:
    image: containous/whoami # A container that exposes an API to show its IP address
    labels:
      - "traefik.frontend.rule=Host:whoami.docker.localhost"
```

```
[root@122-147-213-61 traefik]# docker-compose up -d whoami
Creating traefik_whoami_1 ... done
```
