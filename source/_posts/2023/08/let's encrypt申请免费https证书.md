---
layout: post
title: let's encrypt申请免费https证书
date: 2023-08-08
tags: ["https","letsencrypt","免费证书"]
---

首先要去安装`let's encrypt`使用的客户端`certbot`,

    sudo apt-get update
    sudo apt-get install certbot
<!--more-->

在申请证书的同时,`let's encrypt`会效验域名的合法性,以webroot模式为例,certbot会在-w目录下生成一个`/.well-known/acme-challenge/xxx`的文件,然后调用要生成域名访问这个文件,文件访问成功则认为域名是合法的,所以要先让`certbot`访问到该域名下生成的文件，
修改ngix配置如下：

         server {
            listen 80;
            server_name book.lilliput.fun;
            root /www;  ## 这个目录和certbot中 -w 的目录保存一直
            location ~/.well-known {
                    allow all;
            }
        }

申请证书

    # 这里用webroot模式申请,还有个standalone模式,暂时没用到没研究 -d是跟自己要生成的域名 -w 就是用于放置验证文件的路径
    certbot certonly --webroot -w /www -d book.lilliput.fun

提示这个的时候就申请成功了

    IMPORTANT NOTES:
     - Congratulations! Your certificate and chain have been saved at:
       /etc/letsencrypt/live/book.lilliput.fun/fullchain.pem
       Your key file has been saved at:
       /etc/letsencrypt/live/book.lilliput.fun/privkey.pem
       Your cert will expire on 2023-11-06. To obtain a new or tweaked
       version of this certificate in the future, simply run certbot
       again. To non-interactively renew *all* of your certificates, run
       "certbot renew"
     - If you like Certbot, please consider supporting our work by:

       Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
       Donating to EFF:                    https://eff.org/donate-le

申请完的证书放在了`/etc/letsencrypt/live/域名`下,

然后配置下https

        server {
            server_name book.lilliput.fun;
            listen 443 ssl;
            ssl_certificate /etc/letsencrypt/live/book.lilliput.fun/fullchain.pem;
            ssl_certificate_key /etc/letsencrypt/live/book.lilliput.fun/privkey.pem;
             add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
             location / {
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Upgrade-Insecure-Requests 1;
                proxy_set_header X-Forwarded-Proto https;
                proxy_pass http://book;
            }
        }

最后将80端口重定向到443

       server {
            listen 80;
            server_name book.lilliput.fun;
            location / {
                rewrite ^ https://$host$request_uri? permanent;
            }
        }