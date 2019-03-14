# traefik-nginx-php-note

因為想架的服務大家都想要 :80 跟 :443，但我只有單機電腦，所以必須架一層 proxy manager 來根據 path 來轉到各種服務

[DevOps 台灣](https://t.me/devopstw)推薦了 [traefik](https://traefik.io/) 這個工具覺得不錯

- [traefik-nginx-php-note](#traefik-nginx-php-note)
  - [需求前提](#需求前提)
  - [安裝 traefik](#安裝-traefik)
  - [設置 traefik](#設置-traefik)
  - [安裝 nginx & php](#安裝-nginx--php)
  - [設置 nginx](#設置-nginx)
  - [部屬伺服器](#部屬伺服器)
  - [實驗驗證](#實驗驗證)
  - [Q&A](#qa)

## 需求前提

- 使用 Docker & Compose
- 單機服務(非 cloud)
- 很多服務都想要 :80 :443，而且有很多個 docker-compose.yml (不寫在一起)

## 安裝 traefik

先用 traefik 官方文件的兩張圖來介紹和解釋 traefik 做了什麼

![traefik architecture](traefik-architecture.png)

![traefik internal](traefik-internal.png)

對外讓 :80 :443 接到 traefik，traefik 根據 hostname/path 跟內部 port 接到各個服務

而安裝 traefik 就一行指令：

```sh
$ docker pull traefik
```

## 設置 traefik

可以先看 [appleboy 大大的文章](https://blog.wu-boy.com/2019/01/deploy-service-using-traefik-and-docker/)和[官方文件](https://docs.traefik.io/user-guide/docker-and-lets-encrypt/)知道大概做了什麼

我的設定是偏向官方文件，但我放在 ~/traefik 底下：

```sh
~/traefik
├── acme.json
├── docker-compose.yml
└── traefik.toml
```

因為改了路徑，所以 docker-compose.yml 的 volumes 必須改掉。

```yml
services:
  traefik:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/traefik.toml
      - ./acme.json:/acme.json
```

traefik.toml 則是把 email 改成自己的

```toml
[docker]
endpoint = "unix:///var/run/docker.sock"
watch = true
exposedByDefault = false

[acme]
email = "acme@mail.flandre.tw" # 你的信箱，這邊用我的假信箱舉例
storage = "acme.json"
entryPoint = "https"
onHostRule = true
```

最後把 acme.json 權限改成 600 即可

```sh
~/traefik $ chmod 600 acme.json
~/traefik $ docker network create web # 這裡的 web 對應 docker-compose.yml 的 networks，只有第一次需要
~/traefik $ docker-compose up -d
```

## 安裝 nginx & php

這個網路上已經一堆教學了不多做介紹

```sh
$ docker pull nginx:alpine
$ docker pull php:fpm-alpine
```

## 設置 nginx

因為我的硬碟是 mount 上去的，比較大，故我選擇在這邊設定 (假設 mount 在 `/media/3TB`)

這邊先創三個資料夾

```sh
/media/3TB $ mkdir -p ./web/nginx/conf.d ./web/nginx/logs ./web/www

/media/3TB $ tree web
web # 伺服器設定和部屬的資料夾
├── nginx # 一些 nginx 的設定檔和日誌
│   ├── conf.d
│   └── logs
└── www # 伺服器服務的根
```

創兩個設定檔 web/.env 和 web/docker-compose.yml 給 docker

```env
DOMAIN=flandre.tw
LOCAL_VOLUME=./www/flandre.tw
```

```yml
version: "3.6"

services:
  nginx:
    image: nginx:alpine
    container_name: web-nginx
    restart: unless-stopped
    networks:
      - web # 跟 traefik 同一個，traefik 會幫你做事
      - net-nginx-php # 下面解釋
    depends_on:
      - php
    volumes:
      - "./nginx/conf.d:/etc/nginx/conf.d" # 這邊讓設定檔可以被放進去
      - "./nginx/logs/$DOMAIN:/etc/nginx/logs/$DOMAIN" # 這邊是 domain 的 logs
      - "./nginx/logs:/var/log/nginx/logs" # 這個是 nginx 本身的 logs
      - "$LOCAL_VOLUME:/var/www/$DOMAIN" # 這個是伺服器服務的根，為了讓 nginx 看得到檔案
    labels:
      - "traefik.docker.network=web" # 這個 web 對應上面 create 的名字
      - "traefik.enable=true"
      - "traefik.basic.frontend.rule=Host:$DOMAIN" # 你的 domain
      - "traefik.basic.port=80" # 這邊對應伺服器預設的 port，nginx 是 80
      # 可參考 https://github.com/nginxinc/docker-nginx/blob/master/mainline/alpine/Dockerfile#L142
      - "traefik.basic.protocol=http"
  php:
    image: php:fpm-alpine
    container_name: web-php
    networks:
      - net-nginx-php # 下面解釋
    expose:
      - "9000" # nginx 會用 :9000 呼叫 php 的 fastcgi
      # 可參考 https://github.com/docker-library/php/blob/master/7.3/alpine3.9/fpm/Dockerfile#L226
    volumes:
      - "$LOCAL_VOLUME:/var/www/$DOMAIN" # 這個是伺服器服務的根，為了讓 php 看得到檔案

networks:
  web:
    external: true
  net-nginx-php: # 這個網路只存在在 docker 內部，用來溝通 nginx 和 php
    name: net-nginx-php
```

和另外一個設定檔 web/nginx/conf.d/flandre.tw.conf (這邊叫什麼 .conf 都可以) 給 nginx

```conf
server {
    index index.php index.html;
    server_name flandre.tw; # 你的 domain

    charset utf-8;

    # 下面兩個對應伺服器 docker-compose.yml 的 services.nginx.volumes 的第二項
    error_log  /etc/nginx/logs/flandre.tw/error.log;
    access_log /etc/nginx/logs/flandre.tw/access.log;

    # 下面這個對應伺服器 docker-compose.yml 的 services.nginx.volumes 的第四項
    root /var/www/flandre.tw;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000; # 這邊就是上面說呼叫 fastcgi 的界面
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

最後放個測試頁面 web/www/flandre.tw/index.php

```php
<?php
print "200 ok\n";
?>
```

最後資料夾結構會是這樣：

```sh
/media/3TB $ tree web
web
├── docker-compose.yml
├── .env
├── nginx
│   ├── conf.d
│   │   └── flandre.tw.conf
│   └── logs
└── www
    └── flandre.tw
        └── index.php
```

## 部屬伺服器

```sh
/media/3TB/web $ docker-compose up -d
/media/3TB/web $ curl https://flandre.tw/
200 ok
/media/3TB/web $
```

## 實驗驗證

為了實驗是否可以讓另一個服務聽在 :80 並順利執行

我寫了一個 Dockerfile 並 build 成 image：

```dockerfile
FROM python:3.7-alpine

WORKDIR /root

EXPOSE 80

VOLUME ["/root"]

CMD ["python", "-m", "http.server", "80"]
```

```sh
$ docker build . -t python-http-server
```

另外創個資料夾 /media/3TB/web/w2 寫了個 docker-compose.yml 來起伺服器

```yml
version: "3.6"

services:
  py3-http-server:
    image: py3-http-server
    container_name: py3-http-server
    restart: unless-stopped
    networks:
      - web # 跟 traefik 同一個
    volumes:
      - ".:/root"
    labels:
      - "traefik.docker.network=web"
      - "traefik.enable=true"
      - "traefik.basic.frontend.rule=Host:www.flandre.tw" # 另一個 domain，用 CNAME 指到同一個主機
      - "traefik.basic.port=80" # image 聽 80
      - "traefik.basic.protocol=http"

networks:
  web:
    external: true
```

並寫個 index.html 來區別這個 domain 的服務

```html
418 I'm a teapot
```

完成後資料夾結構會是這樣：

```sh
/media/3TB/web
├── docker-compose.yml # ← nginx
├── nginx
│   ├── conf.d
│   │   └── flandre.tw.conf
│   └── logs
├── w2
│   ├── docker-compose.yml # ← python-http-server
│   └── index.html # ← 418
└── www
    └── flandre.tw # ← nginx root
        └── index.php # ← 200
```

最後在 web/w2 部屬這個小伺服器

```sh
/media/3TB/web/w2 $ docker-compose up -d
```

用 curl 驗證一下

```sh
$ curl -L flandre.tw
200 ok
$ curl -L www.flandre.tw
418 I'm a teapot
```

## Q&A

Q: 為什麼不全部放在 /media/3TB 裡面，要另外把 traefik 的設定放在 ~/traefik ？

```txt
因為我的硬碟是 NTFS，他不給我 chmod，而 acme.json 權限必須是 600
(´・ω・`)
```
