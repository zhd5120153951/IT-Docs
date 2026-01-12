# 解决使用OpenSSL自签名证书访问HTTPS时的不安全警告问题

## *一、证书生成*

### 1、生成包含Subject Alternative Name(SAN)的证书
浏览器(尤其是Chrome)要求证书必须包含**Subject Alternative Name（SAN）**字段，否则即使手动信任也会提示警告。以下是生成符合要求的证书的步骤：

#### 1.1 生成根证书(可选但推荐)
根证书用于签署服务器证书，可简化客户端信任配置。
```bash
# 生成根证书私钥
openssl genrsa -out rootCA.key 2048 (rootCA可以自己命名)

# 生成根证书(有效期10年)
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 \
  -subj "/C=CN/ST=Province/L=City/O=Organization/CN=RootCA" \
  -out rootCA.crt
```

#### 1.2 生成服务器证书
创建配置文件`server.cnf`，定义SAN扩展：
```ini
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=CN
ST=Province
L=City
O=Organization
CN=your.domain.com  # 与SAN中的主域名一致

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = your.domain.com  # 主域名
DNS.2 = www.your.domain.com  # 备用域名
IP.1 = 192.168.1.100  # 服务器IP(可选)
```

生成服务器私钥和证书签名请求（CSR）：
```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config server.cnf
```

使用根证书签署服务器证书(若未生成根证书，可跳过`-CA`参数直接自签名)：
```bash
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key \
  -CAcreateserial -out server.crt -days 3650 -sha256 \
  -extfile server.cnf -extensions req_ext
```

## *2、信任配置*

### 2、手动信任证书(客户端配置)
将根证书(`rootCA.crt`)导入客户端的信任存储：
#### 2.1 Windows
- 运行`certmgr.msc`，进入“受信任的根证书颁发机构”>“证书”。
- 右键选择“导入”，按向导导入`rootCA.crt`。

#### 2.2 macOS
- 打开“钥匙串访问”，选择“系统”钥匙串。
- 拖拽`rootCA.crt`到窗口，双击证书并设置“始终信任”。

#### 2.3 Linux
- 将证书复制到系统信任目录：
  ```bash
  sudo cp rootCA.crt /usr/local/share/ca-certificates/
  sudo update-ca-certificates
  ```

#### 2.4 浏览器
- 在Chrome中，进入`chrome://settings/certificates`，选择“受信任的根证书颁发机构”标签页，点击“导入”。

## *3、服务器设置*
### 3、服务器配置
#### 3.1 Nginx
修改Nginx配置文件(如`/usr/local/nginx/nginx.conf`)：
```nginx
server {
    listen 443 ssl;#端口自定义
    server_name your.domain.com;#填IP即可
    
    ssl_certificate /path/to/server.crt;#服务器证书路径
    ssl_certificate_key /path/to/server.key;
    
    # 启用强加密协议和密码套件
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
}
```

#### 3.2 Apache(和nginx选其一即可)
修改Apache的SSL配置文件(如`/etc/httpd/conf.d/ssl.conf`)：
```apache
<VirtualHost _default_:443>
    ServerName your.domain.com
    SSLCertificateFile /path/to/server.crt
    SSLCertificateKeyFile /path/to/server.key
    SSLCertificateChainFile /path/to/rootCA.crt  # 可选，若使用根证书签署
    
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384
</VirtualHost>
```

## *4、验证与注意事项*
1. **证书有效性检查**：
   ```bash
   openssl x509 -in server.crt -text -noout
   ```
   确认`X509v3 Subject Alternative Name`字段包含正确的域名或IP。

2. **私钥解密**：
   若私钥加密，需在服务器配置中使用解密后的私钥(如`server.key.unsecure`)，避免启动失败。

3. **浏览器兼容性**：
   - Chrome严格要求SAN，必须包含主域名。
   - Firefox和Edge对SAN的要求较宽松，但建议统一配置以避免问题。

4. **生产环境建议**：
   - 优先使用受信任的CA(如Let's Encrypt)颁发的证书，彻底消除警告。
   - 定期更新证书，避免过期。

通过以上步骤，可有效解决自签名证书导致的不安全警告问题。若仍有异常，可检查证书链完整性、服务器配置路径及加密协议设置。