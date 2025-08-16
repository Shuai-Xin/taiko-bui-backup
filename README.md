## 這啥

這是當年紅極一時的 bui 大哥製作的 taiko-web 的原始碼的備份。

雖然萬代已經把大部分服務都 shut down 了，但由於開源的力量，legends nerver die！

## Setup

如今依舊可以按照 setup.md 的步驟，自己在 linux 系統上運行 taiko-web，有能力可以自己搞搞看。

我這邊寫了一份 dockerfile 腳本，其撰寫邏輯有[撰文介紹](https://shuaixin.cc/Docker-Taiko-Web/)，可以在任何安裝了 docker 的 AMD64 架構的系統中快速啟動服務（經測試，win11、ubuntu24 皆可成功運作）：

```
FROM debian:bookworm

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    git python3 python3-pip python3-virtualenv \
    nginx ffmpeg redis-server gnupg\
    supervisor curl && \
    rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
    gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

RUN echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] http://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" \
    | tee /etc/apt/sources.list.d/mongodb-org-8.0.list

RUN apt-get update && \
    apt-get install -y mongodb-org && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /srv/taiko-web /var/log/taiko-web /data/db

RUN git clone https://github.com/Shuai-Xin/taiko-bui-backup.git /srv/taiko-web

RUN cp /srv/taiko-web/config.example.py /srv/taiko-web/config.py

RUN pip3 install --break-system-packages --no-cache-dir -r /srv/taiko-web/requirements.txt

RUN cp /srv/taiko-web/tools/nginx.conf /etc/nginx/conf.d/taiko-web.conf && \
    sed -i 's/include \/etc\/nginx\/sites-enabled\/\*;/#include \/etc\/nginx\/sites-enabled\/\*;/' /etc/nginx/nginx.conf

RUN printf "[supervisord]\nnodaemon=true\n\
[program:mongod]\ncommand=mongod --bind_ip_all\nautorestart=true\n\
[program:redis]\ncommand=redis-server\nautorestart=true\n\
[program:nginx]\ncommand=nginx -g 'daemon off;'\nautorestart=true\n\
[program:taiko_app]\ndirectory=/srv/taiko-web\ncommand=gunicorn -b 127.0.0.1:34801 app:app\nautorestart=true\n\
[program:taiko_server]\ndirectory=/srv/taiko-web\ncommand=python3 server.py 34802\nautorestart=true" > /etc/supervisord.conf

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```
