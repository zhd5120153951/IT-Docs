# docker 部署 nginx

## 镜像准备
```text
下载 nginx 镜像
docker pull nginx:1.25~1.29
```
## 配置文件准备
```bash
一般在/etc/nginx/conf.d/default.conf中配置反向代理规则；
但是建议新建一个配置site-enabled目录，将反向代理规则配置在site-enabled/default.conf中或者新建reverse-proxy.conf文件
server {
    listen 8443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/cert/server.crt;
    ssl_certificate_key /etc/nginx/cert/server.key;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location /api/ {
        proxy_pass http://127.0.0.1:8001/;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /ai/ {
        proxy_pass http://127.0.0.1:8002/;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## 负载均衡配置

```bash
示例1：最简单的轮询负载均衡（HTTP）
upstream backend {
    server 10.0.0.11:8000;
    server 10.0.0.12:8000;
    server 10.0.0.13:8000;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
示例2：带权重、失败检测与keepalive（生产级）
upstream backend {
    server 10.0.0.11:8000 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:8000 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.0.13:8000 weight=1 max_fails=3 fail_timeout=30s;

    keepalive 32;  # 后端长连接池，减少握手开销
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 调整超时以防长请求被切断
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
示例3：HTTPS终端+将流量分发到两个内网服务（你之前提到的8001/8002）
按路径路由到不同后端（单8443端口同时代理两个服务）：
server {
    listen 8443 ssl;
    server_name _;

    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # 通用头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location /api/ {
        proxy_pass http://127.0.0.1:8001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /ai/ {
        proxy_pass http://127.0.0.1:8002/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
示例4：WebSocket（多后端）负载均衡
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream ws_backend {
    least_conn;
    server 10.0.0.11:9001;
    server 10.0.0.12:9001;
}

server {
    listen 80;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;  # 长连接可能需要更长超时
    }
}
```