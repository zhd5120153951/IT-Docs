# 大型项目的nginx代理指南

## 1、代码层级如下
/usr/local/nginx/conf
│
├── nginx.conf
│
├── upstreams
│   ├── ai.conf
│   ├── asr.conf
│   └── tts.conf
│
├── servers
│   ├── api.conf
│   ├── websocket.conf
│   └── monitor.conf
│
└── ssl
详细配置如下:
```nginx
1. nginx.conf
http {

    include mime.types;

    include upstreams/*.conf;

    include servers/*.conf;

}
2. upstreams/ai.conf
upstream ai_backend {

    server 127.0.0.1:8001;
    server 127.0.0.1:8002;

}
3. servers/api.conf
server {

    listen 443 ssl;

    ssl_certificate ssl/server.crt;
    ssl_certificate_key ssl/server.key;

    location / {

        proxy_pass http://ai_backend;

    }
}
```
## 2、动态反向代理(大型AI平台常用)
```nginx
upstream asr_backend {

    server 10.0.0.10:9001;
    server 10.0.0.11:9001;

}

upstream tts_backend {

    server 10.0.0.20:9002;
    server 10.0.0.21:9002;

}

location /asr/ {
    proxy_pass http://asr_backend;
}

location /tts/ {
    proxy_pass http://tts_backend;
}

```
## 后续扩容方便
```nginx
upstream tts_backend {

    server 10.0.0.20:9002;
    server 10.0.0.21:9002;
    server 10.0.0.22:9002;
    server 10.0.0.23:9002;

}
```