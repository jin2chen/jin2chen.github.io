[TOC]

# Ningx 2-way SSL Verify

## 生成证书
### 使用openssl自带脚本生成 _CA_, _server_, _client_ 证书
具体生成请参考下面脚本  
**注意：** 生成 server 证书时，根据实际域名修改强调部分 '/C=US/ST=hunan/L=changsha/O=SI Co.,Ltd./OU=Server/**CN=a.si.dev**/emailAddress=server@si.com'

```SHELL
#!/bin/sh
rm -rf *.crt *.csr *.key *.pem demoCA
mkdir -p ./demoCA
mkdir -p ./demoCA/certs
mkdir -p ./demoCA/crl
mkdir -p ./demoCA/newcerts
mkdir -p ./demoCA/private
touch ./demoCA/index.txt

openssl req -nodes -new -keyout CA.key -out CA.csr -subj '/C=CN/ST=hunan/L=changsha/O=SI Co.,Ltd./OU=CA/CN=CA inc./emailAddress=ca@si.com'
openssl ca -create_serial -out CA.crt -days 1095 -batch -keyfile CA.key -selfsign -extensions v3_ca -infiles CA.csr
openssl req -new -nodes -out server.csr -keyout server.key -subj '/C=US/ST=hunan/L=changsha/O=SI Co.,Ltd./OU=Server/CN=a.si.dev/emailAddress=server@si.com'
openssl ca -cert CA.crt -keyfile CA.key -policy policy_anything -in server.csr -out server.crt -batch
cat server.crt server.key > server.pem
openssl req -new -nodes -out client.csr -keyout client.key -subj '/C=CN/ST=hunan/L=changsha/O=SI Co.,Ltd./OU=Client/CN=USER001/emailAddress=user@si.com'
openssl ca -cert CA.crt -keyfile CA.key -in client.csr -out client.crt -batch
cat client.crt client.key > client.pem
```

## 配置nginx
请替换 **PATH** 为证书真实路径，重启 nginx service.

```NGINX
server {
   listen 443;
   ssl                        on;
   ssl_certificate            /PATH/server.crt;
   ssl_certificate_key        /PATH/server.key;
   ssl_client_certificate     /PATH/CA.crt;
   ssl_session_timeout        5m;
   ssl_verify_client          on;
   ssl_protocols              SSLv2 SSLv3 TLSv1;
   ssl_ciphers                ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
   ssl_prefer_server_ciphers  on;

   server_name a.si.dev;
   root        /app/rest/advisor/web;
   index       index.php;

   location / {
       # Redirect everything that isn't a real file to index.php
       try_files $uri $uri/ /index.php?$args;
   }

   # uncomment to avoid processing of calls to non-existing static files by Yii
   location ~ \.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar)$ {
       try_files $uri =404;
   }

   location ~ \.php$ {
       include fastcgi_params;
       fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
       fastcgi_pass unix:/run/php/php5.6-fpm.sock;
       try_files $uri =404;
   }

   location ~ /\.(ht|svn|git) {
       deny all;
   }
}

```

## 测试

```SHELL
curl --cacert CA.crt --cert client.pem https://a.si.dev/v1/applications
```
--cacert 指定根证书，以验证 nginx server 的证书  
--cert 指定请求证书，让 nginx server 验证

## 使用 GuzzleHttp 请求
根据证书实际路径替换 **verify, cert** 参数

```PHP
$client = new \GuzzleHttp\Client([
    'debug' => true,
    'timeout' => 60,
    'http_errors' => false,
    'verify' => '/app/vagrant/nginx/ca/CA.crt',
    'cert' => '/app/vagrant/nginx/ca/client.pem',
]);

$response = $client->get('https://a.si.dev/v1/applications');
echo $response->getBody();
echo "\n";
```
