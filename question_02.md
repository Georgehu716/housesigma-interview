二、根据要求提供一份Nginx配置：

    域名：ipo.com, 支持https、HTTP/2
    非http请求经过301重定向到https
    根据UA进行判断，如果包含关键字 "Google Bot", 反向代理到 server_bot[bot.ipo.com] 去处理
    /api/{name} 路径的请求通过unix sock发送到本地 php-fpm，文件映射 /www/api/{name}.php
    /api/{name} 路径下需要增加限流设置，只允许每秒1.5个请求，超过限制的请求返回 http code 429
    /static/ 目录下是纯静态文件
    其它请求指向目录 /www/ipo/, 查找顺序 index.html --> public/index.html --> /api/request


Solutions:

`nginx.conf`

```
server {
    listen 80;
    server_name ipo.com;

    # 非 HTTP 请求重定向到 HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ipo.com;

    # SSL 整数配置
    ssl_certificate /etc/ssl/certs/ipo.com.cert;
    ssl_certificate_key /etc/ssl/private/ipo.com.key;

    # 根据 User-Agent 反向代理到 server_bot
    if ($http_user_agent ~* "Google Bot") {
        return 301 https://bot.ipo.com$request_uri;
    }

    # 静态文件配置
    location /static/ {
        root /www/ipo;
        access_log off;
        expires max;
    }

    # 限制 /api/{name} 路径的请求速率
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=1.5r/s;

    # 处理 /api/{name} 请求，传递给 php-fpm
    location ~ ^/api/(.*)$ {
        limit_req zone=api_limit burst=10 nodelay;
        error_page 429 = @limit_reached;

        root /www/api;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME /www/api/$1.php;
        include fastcgi_params;
    }

    location @limit_reached {
        return 429 "Too Many Requests";
    }

    # 其它请求，查找顺序: index.html --> public/index.html --> /api/request
    location / {
        root /www/ipo;
        index index.html;

        try_files $uri $uri/ /public/index.html /api/request;
    }
}
```