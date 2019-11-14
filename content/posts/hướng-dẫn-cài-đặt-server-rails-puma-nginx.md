---
title: "Cài Đặt Server Chạy Rails App Trên Production Sử Dụng Puma + Nginx"
date: 2019-11-12T15:24:47+09:00
draft: no
tags: [infra, rails, tech, nginx, deploy]
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/puma-nginx.png
---

**Puma là web server nhỏ gọn đi liền trong Rails giúp developer có thể bắt đầu code một cách nhanh nhất. Tuy nhiên mang nó làm web server thực sự để chạy trên môi trường production thì chưa ổn. Bài này mình muốn memo lại chia sẻ với các bạn các bước cài đặt server để ứng dụng Rails chạy ổn định trên môi trường production.**

### Prerequisite
- OS môi trường là Amazon linux
- Database sử dụng là Postgres

### System configuration

Puma hoạt động như application server cho ứng dụng Rails, 
còn Nginx sẽ hoạt động với vai trò là reverse proxy - nhận request và chuyển response giữa client và Puma.
Puma và Nginx giao tiếp với nhau thông qua socket.

- rbenv + Ruby 2.5
- Rails 5 + Puma + Nginx

![puma-nginx.png](/blog/images/puma-nginx.png)

> *Source Image : http://codeonhill.com*

**Ok, ssh vào server và bắt đầu cài đặt thôi! ↓↓↓**

---

### 1. Chuẩn bị môi trường

```
# Cài đặt các package cơ bản
# ※ Riêng cái htop thì optional, mình thích dùng htop để xem trạng thái server nên có thói quen cài nó
$ sudo yum update
$ sudo yum install \
git make gcc-c++ patch \
openssl-devel \
libyaml-devel libffi-devel libicu-devel \
libxml2 libxslt libxml2-devel libxslt-devel \
zlib-devel readline-devel \
postgresql-libs postgresql-devel \
epel-release htop
$ sudo amazon-linux-extras install epel
$ sudo yum install nodejs npm --enablerepo=epel
```

### 2. Tạo user
Thói quen tốt là không nên dùng user root/ec2-user mặc định, do đó ta sẽ tạo một user
mới để phục vụ quá trình cài đặt cũng như khởi chạy ứng dụng Rails.

```
# Tạo user deploy
$ sudo adduser deploy
$ sudo passwd deploy  # Cài đặt password cho user deploy
$ sudo visudo         # Gán quyền cho user deploy
-----------------------------
# vim khởi động lên, thêm quyền bằng cách thêm dòng sau
root    ALL=(ALL)       ALL
deploy  ALL=(ALL)       ALL # ← thêm dòng này
-----------------------------
$ sudo su - deploy    # Chuyển qua user deploy
# Cài đặt timezone
$ sudo ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
# Cài đặt alias ll, hữu dụng và dễ đọc file size hơn lệnh ls
$ echo 'alias ll="ls -lh --color=auto"' >> ~/.bash_profile
$ source ~/.bash_profile
```

### 3. Cài đặt rbenv và Ruby
Sử dụng rbenv để quản lý các version của ruby.

```
# Cài đặt rbenv
$ sudo su - deploy
$ git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
$ source ~/.bash_profile
$ git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
$ rbenv rehash
# Cài đặt ruby 2.5
$ rbenv install -v 2.5.1
$ rbenv global 2.5.1
$ rbenv rehash
$ ruby -v
```

### 4. Chuẩn bị cho ứng dụng Rails

```
# Lấy source của ứng dụng về
$ sudo mkdir -p /var/www
$ sudo chown -R deploy:deploy /var/www
$ sudo chmod 775 /var/www/
$ cd /var/www
$ git clone abc@example.com:/great_app.git && cd great_app
# bundle install
$ gem install bundler -v 1.16.6 # check bundler's version in Gemfile.lock
$ bundle i
$ source ~/.bash_profile

# Phải generate mới credential cho Rail vì những file này bị ignore khỏi git repository
$ rm config/credentials.yml.enc
$ rm config/master.key
$ EDITOR=vim bin/rails credentials:edit

# Tạo file .env vì file này cơ bản cũng bị ignore khỏi git repository
$ touch .env
$ vi .env    # Cài đặt các biến môi trường bạn cần dùng

# Precompile
$ RAILS_ENV=production rails assets:precompile
```

### 5. Cài đặt Puma chạy dạng service - chạy ở background và tự khởi chạy

```
# Đầu tiên cần tạo file cấu hình cho service puma
$ sudo touch /etc/systemd/system/puma.service
# Nội dung file puma.service tham khảo mình để ở dưới
$ sudo vi /etc/systemd/system/puma.service
$ sudo systemctl daemon-reload

# Cho phép chạy puma khi boot
$ sudo systemctl enable puma
# Khởi động puma
$ sudo systemctl restart puma

# Nếu cần khởi động puma bằng tay, chạy lệnh sau
$ RAILS_ENV=production /home/deploy/.rbenv/shims/puma -C /var/www/great_app/config/puma.rb
```

*File puma.service tham khảo*

```
[Unit]
Description=Puma HTTP Server
After=network.target

# Uncomment for socket activation (see below)
# Requires=puma.socket

[Service]
# Foreground process (do not use --daemon in ExecStart or config.rb)
Type=simple

# Preferably configure a non-privileged user
# User=

# The path to the your application code root directory.
# Also replace the "<YOUR_APP_PATH>" place holders below with this path.
# Example /home/username/myapp
WorkingDirectory=/var/www/great_app

# Helpful for debugging socket activation, etc.
# Environment=PUMA_DEBUG=1

# SystemD will not run puma even if it is in your path. You must specify
# an absolute URL to puma. For example /usr/local/bin/puma
# Alternatively, create a binstub with `bundle binstubs puma --path ./sbin` in the WorkingDirectory
Environment=RAILS_ENV=production
ExecStart=/home/deploy/.rbenv/shims/puma -C /var/www/great_app/config/puma.rb

Restart=always

[Install]
WantedBy=multi-user.target
```

### 6. Cài đặt Nginx

```
# cài nginx
$ sudo yum -y install nginx
# Nội dung file greatapp.nginx.conf tham khảo mình để ở dưới
$ sudo vi /etc/nginx/conf.d/greatapp.nginx.conf

# Cho phép chạy nginx khi boot
$ sudo systemctl enable nginx
# Khởi động nginx
$ sudo systemctl restart nginx
$ sudo chown -R nginx:nginx /var/lib/nginx/ # 1回権限エラーで困った為、このフォルダーの権限を変更しておく

# Thay đổi quyền cho log folders
$ sudo chown -R deploy:deploy /var/log/nginx
$ sudo chown -R deploy:deploy /var/log/nginx/*
$ sudo chown -R deploy:deploy /var/www/great_app/log/*
```

*File greatapp.nginx.conf tham khảo*

```
# log directory
error_log  /var/www/great_app/log/nginx.error.log;
access_log /var/www/great_app/log/nginx.access.log;

upstream app_server {
    # for UNIX domain socket setups
    server unix:///var/www/great_app/puma/puma.sock fail_timeout=0;
}

server {
    listen 80;
    server_name great-app.com
                *.great-app.com;

    # path for static files
    root /var/www/great_app/public;

    # page cache loading
    try_files $uri/index.html $uri @app_server;

    location / {
        # If requested files exists serve them
        try_files $uri $uri @app;
    }

    location @app {
        # When nginx should return maintenance page?
        # - when tmp/maintenance.txt file exists
        if (-f $document_root/../tmp/maintenance.txt) {
            set $maint_mode 1;
        }
        if ($remote_addr = 127.0.0.1) {
            set $maint_mode 0;
        }
        if ($maint_mode) {
            return 503;
        }

        # HTTP headers
        proxy_pass http://app_server;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
    }

    # Maintenance page
    error_page 503 /maintenance.html;
    location /maintenance.html {
        if (!-f $request_filename) {
            rewrite ^(.*)$ /maintenance.html break;
        }
    }

    # Error pages
    error_page 500 502 504 /500.html;
    location = /500.html {
        root /var/www/great_app/public;
    }
    error_page 404 /404.html;
    location = /404.html {
        root /var/www/great_app/public;
    }

    client_max_body_size 4G;
    keepalive_timeout 10;
}
```

### 7. Cài đặt log rotation
Log của nginx, puma, rails theo thời gian sẽ tăng dần do đó cài đặt để rotation log
là thói quen chúng ta nên làm.

```
# Nội dung file greatapp.logrotate.conf tham khảo mình để ở dưới
$ sudo vi /etc/logrotate.d/greatapp.logrotate.conf
```

*File greatapp.logrotate.conf tham khảo*

```
# Nginx log
/var/log/nginx/*.log
{
    missingok
    weekly       # rotate hàng tuần
    rotate 12    # keep đến 12 bản log gần đây nhất
    dateext
    dateformat -%Y%m%d
}

# Application log
/var/www/great_app/log/*.log
{
    missingok
    weekly
    rotate 12
    dateext
    dateformat -%Y%m%d
}
```
---