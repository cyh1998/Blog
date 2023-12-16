#### 配置SSL
```
server {
    # 配置默认访问443端口
    listen       443 ssl;
    # 配置证书对应域名
    server_name  your_domain_name;

    # 配置证书文件
    ssl_certificate     certificate_file.crt;
    # 配置证书私钥文件
    ssl_certificate_key certificate_key_file.key;
    # 配置TLS协议类型
    ssl_protocols       TLSv1.2;
    # 配置加密算法类型
    ssl_ciphers         ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;

    # ...
}
```

#### 配置无后缀访问
```
server {
    location /your_path{
        try_files $uri $uri/ $uri.php$is_args$args;
    }

    # ...
}
```

#### 配置wss协议
游戏中使用wss协议，而wss协议必须使用域名进行访问。  
当客户端可以访问多个游戏服务器时，按照IP+端口的方式，那么每一台服务器都需要申请域名，且同时还要维护每台机器上的证书。  
此时，我们可以用Nginx做代理，在一台服务器(想要高可用可以用两台)上即可用一个域名访问多个不同游戏服务器。  
配置如下：
```
server {
    location /wss {
        proxy_pass http://$arg_ip:$arg_port;
        proxy_read_timeout 60s;
        proxy_http_version 1.1;

        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }

    # ...
}
```
SSL/TLS的配置可以见上文，这里不再展示。  
Nginx将客户端的wss协议处理为ws协议发给服务器，将服务器的ws协议处理为wss协议发给客户端。而ws协议是支持用IP+端口访问的。  
这样，客户端只需要访问：`域名/wss/?ip=xxx&port=xxx` 即可连接对应不同的游戏服务器。
