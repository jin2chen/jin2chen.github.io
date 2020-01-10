# Nginx Dynamic Root Config

### Yii2 implement
```
http://yii2-test-frontend.app  
http://yii2-test-frontend.app/backend/
http://yii2-test-frontend.app/admin/
```

```nginx
server {
    listen 80;
    listen 443 ssl;
    ssl_certificate     /etc/nginx/ssl/yii2-test-frontend.app.crt;
    ssl_certificate_key /etc/nginx/ssl/yii2-test-frontend.app.key;
    server_name yii2-test-frontend.app;
    index index.html index.htm index.php;
    sendfile off;
    charset utf-8;
    client_max_body_size 100m;

    error_log /home/vagrant/nginx-error.log debug_http;
    access_log off;
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    set $root "/home/vagrant/workspace/local/yii2-test/frontend/web";
    set $orgscript "";

    if ($request_uri ~* ^/admin/) {
        set $root "/home/vagrant/workspace/local/yii2-test/admin";
        set $orgscript "/admin";
        rewrite ^/admin/(.*) /$1 last;
        break;
    }

    if ($request_uri ~* ^/backend/) {
        set $root "/home/vagrant/workspace/local/yii2-test/backend/web";
        set $orgscript "/backend";
        rewrite ^/backend/(.*) /$1 last;
        break;
    }

    root $root;
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        #try_files      $uri =404;
        include        fastcgi_params;
        fastcgi_pass   unix:/var/run/php/php7.0-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param  SCRIPT_NAME $orgscript$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
