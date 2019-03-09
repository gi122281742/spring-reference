# 域名HTTPS

## 1.申请证书
由于是腾讯云服务器，首先直接在SSL管理页面申请SSL证书。
申请之后下载证书

## 2.配置NGINX
安装nginx需要加入SSL模块
```
./configure \
--user=dd \
--group=dd \
--prefix=/usr/local/nginx \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-http_ssl_module \
--with-threads
```
设置NGINX.CONF文件
```
server {
        listen 80;
        listen 443 ssl;
        server_name www.ddota.cn; #填写绑定证书的域名
    #   ssl on;
        ssl_certificate 1_www.ddota.cn_bundle.crt;
        ssl_certificate_key 2_www.ddota.cn.key;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #按照这个协议配置
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;#按照这个套件配置
        ssl_prefer_server_ciphers on;
        location / {
            root   html; #站点目录
            index  index.html index.htm;
        }
    }
```
```
./nginx -s reload
```
重启OK。