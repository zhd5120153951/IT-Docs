# Ubuntu系统上完成 Nginx 1.26.2 的源码编译安装全过程，并配置 systemd 开机自启、权限控制、状态监控等关键功能

nginx核心特性如下：
![alt text](images/image.png)

1. 更新软件包&安装编译依赖
```bash
sudo apt-get update
sudo apt-get upgrade 
sudo apt-get install -y build-essential libpcre3-dev libssl-dev zlib1g-dev curl wget vim

```
2. 下载并解压nginx源码包
```bash
curl -O https://nginx.org/download/nginx-1.26.2.tar.gz    # 下载 Nginx 1.26.2 源码包
tar -vxzf nginx-1.26.2.tar.gz                             # 解压,压缩vczf
cd nginx-1.26.2                                           # 进入目录
```
3. 编译与安装
```bash
./configure \
--prefix=/usr/local/nginx-1.26.2 \
# --user=nginx \ #可选参数,为了避免被攻击不要使用master作为启动用户
# --group=nginx \#可选参数
--with-http_ssl_module \
--with-http_v2_module \
--with-http_stub_status_module \
--with-stream \
--with-http_gzip_static_module \
--with-pcre
```
```bash
sudo make && sudo make install
```
4. 创建专门运行用户(可选)
```bash
sudo useradd -r -s /sbin/nologin -M nginx 
# -r：创建系统用户（无登录 shell）
# -s /sbin/nologin：禁止该用户登录系统
# -M：不创建家目录，节省空间
```
设置目录权限：
```bash
sudo chown -R nginx:nginx /usr/local/nginx*
```
5. 配置 systemd 开机自启
```bash
sudo vim /etc/systemd/system/nginx.service

[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
ExecStartPre=/usr/bin/rm -f /run/nginx.pid
ExecStartPre=/usr/local/nginx-1.26.2/sbin/nginx -t
ExecStart=/usr/local/nginx-1.26.2/sbin/nginx
ExecReload=/usr/local/nginx-1.26.2/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT 
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
```
6. 启动 Nginx 并设置开机自启
```bash
sudo systemctl daemon-reload    # 重新加载 systemd 配置
sudo systemctl start nginx      # 启动服务
sudo systemctl enable nginx     # 设置开机自启
sudo systemctl status nginx     # 查看运行状态
#备注：编译安装后的nginx.conf默认在/usr/local/nginx-1.26.2/conf/nginx.conf
#如果需要自定义代理的接口页面，需要在此处修改后，重启nginx服务
sudo systemctl restart nginx    #重启nginx
```
7. 常用命令和错误
```bash
sudo /usr/local/nginx/sbin/nginx	手动启动 Nginx
sudo /usr/local/nginx/sbin/nginx -s stop	立即停止（暴力）
sudo /usr/local/nginx/sbin/nginx -s quit	优雅关闭（处理完请求）
sudo /usr/local/nginx/sbin/nginx -s reload	重载配置（不中断服务）
sudo /usr/local/nginx/sbin/nginx -t	测试配置文件语法
sudo systemctl restart nginx	重启服务（systemd）
```

```bash
C compiler cc not found	未安装编译器	sudo apt install build-essential
Address already in use: 80	端口被占用	sudo netstat -tulnp | grep :80 查看并 kill
Permission denied	权限不足	sudo chown -R nginx:nginx /usr/local/nginx*
nginx.service not found	服务文件路径错误	检查 /etc/systemd/system/nginx.service 是否存在
```