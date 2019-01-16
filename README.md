# advanced-nginx-cheatsheet

**Work In Progress, more explanations will be added soon**

**Table of content**

[TOC]

## SEO

### robots.txt location

```nginx
location = /robots.txt {
# Some WordPress plugin gererate robots.txt file
# Refer #340 issue
    try_files $uri $uri/ /index.php?$args @robots;
    access_log off;
    log_not_found off;
}
location @robots {
    return 200 "User-agent: *\nDisallow: /wp-admin/\nAllow: /wp-admin/admin-ajax.php\n";
}
```

### Make a website not indexable

```nginx
add_header X-Robots-Tag "noindex";

location = /robots.txt {
  return 200 "User-agent: *\nDisallow: /\n";
}
```

## Media

### MP4 stream module

```nginx
location /videos {
    location ~ \.(mp4)$ {
        mp4;
        mp4_buffer_size       1m;
        mp4_max_buffer_size   5m;
        add_header Vary "Accept-Encoding";
        add_header "Access-Control-Allow-Origin" "*";
        add_header Cache-Control "public, no-transform";
        access_log off;
        log_not_found off;
        expires max;
    }
}
```

### WebP images

```nginx
# webp rewrite rules for jpg and png images
# try to load alternative image.png.webp before image.png
location /wp-content/uploads {
    location ~ \.(png|jpe?g)$ {
        add_header Vary "Accept-Encoding";
        add_header "Access-Control-Allow-Origin" "*";
        add_header Cache-Control "public, no-transform";
        access_log off;
        log_not_found off;
        expires max;
        try_files $uri$webp_suffix $uri =404;
    }
}
```

## Load-Balancing

### php-fpm Unix socket

```nginx
upstream php {
    least_conn;
    server unix:/var/run/php-fpm.sock;
    server unix:/var/run/php-two-fpm.sock;
    keepalive 5;
}
```

### php-fpm TCP

```nginx
upstream php {
    least_conn;
    server 127.0.0.1:9090;
    server 127.0.0.1:9091;
    keepalive 5;
}
```

### HTTP load-balancing

```nginx
# Upstreams
upstream backend {
    least_conn;

    server 10.10.10.1:80;
    server 10.10.10.2:80;
}

server {

    server_name site.ltd;

    location / {
        proxy_pass http://backend;
        proxy_redirect      off;
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## Proxy

### Simple Proxy

```nginx
location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_redirect      off;
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }
```

### Proxy in a subfolder

```nginx
location /folder/ { # The / is important!
        proxy_pass http://127.0.0.1:3000/;# The / is important!
        proxy_redirect      off;
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }
```

### Proxy keepalive for websocket

```nginx
# Upstreams
upstream backend {
    server 127.0.0.1:3000;
    keepalive 5;
}
# HTTP Server
server {
    server_name your_hostname.com;
    error_log /var/log/nginx/rocketchat.access.log;
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forward-Proto http;
        proxy_set_header X-Nginx-Proxy true;
        proxy_redirect off;
    }
}
```

## Security

### Deny access

#### common backup and archives files

```nginx
location ~* "\.(old|orig|original|php#|php~|php_bak|save|swo|aspx?|tpl|sh|bash|bak?|cfg|cgi|dll|exe|git|hg|ini|jsp|log|mdb|out|sql|svn|swp|tar|rdf)$" {
    deny all;
}
```

#### Deny access to hidden files & directory

```nginx
location ~ /\.(?!well-known\/) {
    deny all;
}
```

### Blocking common attacks

#### base64 encoded url

```nginx
location ~* "(base64_encode)(.*)(\()" {
    deny all;
}
```

#### javascript eval() url

```nginx
location ~* "(eval\()" {
    deny all;
}
```

