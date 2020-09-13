# Laravel-base
個人的に一番しっくりくるLaravel開発環境です。
DockerでNginx、PHP-FPM、MySQLを構成しています。
ペースにしているのは下記サイトに記載されていた方法です。
ディレクトリやPHPのバージョンが違いますが、ほぼ同じやり方で記載しています。

参考：
 [[DigitalOcean - How To Set Up Laravel, Nginx, and MySQL with Docker Compose:https://www.digitalocean.com/community/tutorials/how-to-set-up-laravel-nginx-and-mysql-with-docker-compose]] 
 
## Dockerのインストール
### インストール
以下はUbuntu18.04での作業になります。Macの人はDocker for Macを、Windowsの人はDocker for Windowsをインストールして、「LaravelのベースプロジェクトをGitから取得する」以降を実施してください。
```
# 旧バージョンのDockerをアンインストール
$ sudo apt-get remove docker docker-engine docker.io

# aptのアップデート
$ sudo apt-get update

# HTTPSリポジトリを利用できるようにする
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common

# GPGキーの追加
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# DockerリポジトリをAPTソースに追加
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# インストール
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-compose
```
### インストール後の処理
一般ユーザーでDockerを実行できるようにする
```
# ユーザーをdockerグループに追加
$ sudo groupadd docker
$ sudo usermod -aG docker $USER

# テスト
$ docker run hello-world
```
### エラーの一例
下記エラーが出る場合（idコマンドなどでdockerグループが表示されない場合です）があります。
```
docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.39/containers/create: dial unix /var/run/docker.sock: connect: permission denied.
See 'docker run --help'.
```
dockerグループへログインして有効にします。
```
$ newgrp docker

# プライマリのグループがdockerになるので、ユーザーのグループに戻す（hogeユーザーのhogeグループの場合）
$ newgrp hoge
```

## LaravelのベースプロジェクトをGitから取得する
composerなどで作成してもよいですが、ここではgitから取得する方法で作成します。
また、ホームディレクトリにsrc/laravel-baseというディレクトリを作ることを前提にしています。適宜変更してください。

```
$ mkdir -p ~/src/laravel-base
$ cd ~/src/laravel-base

# laravleの最新版をclone
$ git clone https://github.com/laravel/laravel.git

# 特定のバージョンを指定する場合は、cloneする際にバージョンを指定します。
$ git clone -v 5.5.28 https://github.com/laravel/laravel.git
```

## Composer
```
$ cd ~/src/laravel-base/laravel
$ docker run --rm -v $(pwd):/app composer install
```
docker runコマンドに--vオプションをつけることで現在のディレクトリをバインドマウントしたコンテナが作成されます。Windowsの人は「$(pwd)」のところに直接パスを書きます。--rmはDocker終了後に削除するオプションです。別にこの段階やらなくても大丈夫です。

Linuxの人はアクセス権の問題を回避するため、所有者をroot以外のユーザーに設定します。
```
$ sudo chown -R $USER:$USER ~/src/laravel-base
```

## Docker Composeファイルの作成
mroongaのDockerイメージはrootパスワードやdatabaseを自動作成しませんが、ここではMySQLのDockerイメージを使用する場合に備えて記載しています。
PHPはDockerfileで指定するため、後ほど作成します。
```
$ vim docker-compose.yml
```
```
version: '3'
services:

  #PHP Service
  app:
    build:
      context: .
      dockerfile: ./php/Dockerfile
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./laravel:/var/www
      - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - app-network

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./laravel:/var/www
      - ./nginx/conf.d/:/etc/nginx/conf.d/
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:8
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_USER: laravel
      MYSQL_PASSWORD: password
      TZ: 'Asia/Tokyo'
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - dbdata:/var/lib/mysql/
      - ./mysql/log:/var/log/mysql
      - ./mysql/my.cnf:/etc/mysql/my.cnf
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge
#Volumes
volumes:
  dbdata:
    driver: local
```

## Dockerfileの作成
PHP7.4-FPM用のDockerfileを作成します。
```
$ vim ~/src/laravel-base/php/Dockerfile
```
```
FROM php:7.4-fpm

# Copy composer.lock and composer.json
COPY ./laravel/composer.lock /var/www/
COPY ./laravel/composer.json /var/www/

# Set working directory
WORKDIR /var/www

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    libonig-dev \
    curl \
    libzip-dev \
    zlib1g-dev

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# node.js
RUN curl -sL https://deb.nodesource.com/setup_lts.x | bash -
RUN apt-get install -y nodejs

# Install extensions
RUN docker-php-ext-install pdo_mysql zip exif pcntl
RUN docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/
RUN docker-php-ext-install gd

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy existing application directory contents
COPY . /var/www

# Copy existing application directory permissions
COPY --chown=www:www . /var/www

# Change current user to www
USER www

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```

## PHPの設定
PHP7.2-FPMの設定です。phpディレクトリを作成して、その配下にiniファイルを設置します。
```
$ mkdir ~/src/laravel-base/php
$ vim ~/src/laravel-base/php/local.ini
```
```
upload_max_filesize=40M
post_max_size=40M
date.timezone = "Asia/Tokyo"
```

## Nginxの設定
Nginxのコンフィグファイルを作成します。80版ポートで動作させてます。
```
$ mkdir -p ~/src/laravel-base/nginx/conf.d
$ vim ~/src/laravel-base/nginx/conf.d/app.conf
```
```
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```
動作時にタイムアウトが問題になる場合は下記 keepalive_timeout、send_timeout、fastcgi_read_timeout、fastcgi_connect_timeout、fastcgi_send_timeoutオプションを指定する（未検証。増やしすぎるとパフォーマンスの劣化につながるため注意）
```
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
    keepalive_timeout 600;
    send_timeout 600;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_read_timeout 600;
        fastcgi_connect_timeout 600;
        fastcgi_send_timeout 600;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

## MySQLの設定
```
$ mkdir ~/src/laravel-base/mysql
$ vim ~/src/laravel-base/mysql/my.cnf
```
general_logは状況によって。
```
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_bin
general_log = 0
general_log_file = /var/lib/mysql/general.log
log_timestamps=SYSTEM
secure-file-priv=NULL
default-authentication-plugin = mysql_native_password

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```
ログ用のディレクトリも作成しておきます。
```
$ mkdir ~/src/laravel-base/mysql/log
```

## Laravelの環境設定ファイルの作成
Laravelの環境設定ファイルを作成します。環境設定ファイル（.env）はgitの管理下からは外しますので、本番環境では別途用意します。
```
$ cp .env.example .env
$ vim .env
```
laravel-baseをgit cloneした場合は下記を参考にしてください。DB_HOSTにはデータベースのコンテナ名を記載します。
```
APP_NAME=Laravel
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=null
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"

```

## Gitの管理下に含めないファイル
Docker用のファイルなど、アプリケーションに直接関係のないファイルはGitの管理下には置かないようにします。
```
$ vim .gitignore
```
```
/node_modules
/public/hot
/public/storage
/storage/*.key
/vendor
.env
.phpunit.result.cache
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log
docker-compose.yml
Dockerfile
```

## パッケージのインストール
下記を実行して、venderパッケージをインストールします。
```
$ docker-compose exec app php composer install
```

## Dockerコンテナの作成とLaravelの起動時の設定
コンテナの作成と起動を実行します。
```
$ docker-compose up -d
```
作成が終わったら、動作しているか確認します。
```
# 起動中のコンテナの確認
$ docker ps
```
Laravelのアプリケーションキーを生成します。
```
$ docker-compose exec app php artisan key:generate
```
設定をキャッシュする場合は以下を実行します。設定内容が/var/www/bootstrap/cache/config.phpコンテナにロードされます。
```
$ docker-compose exec app php artisan config:cache
```
マイグレーションを実行し、データベース無いに認証機能に必要なテーブルを作成
```
$ docker-compose exec app php artisan migrate
```
設定が完了したら、下記URLで確認します。
http://localhost
終了する場合は下記を実行します。
```
$ docker-compose down

# volumeごと消すとき（MySQLがうまく設定できないとか。データが消えるので注意）
docker-compose down --volumes
```
不調のときに試すもの。データが消えたりするので注意
```
# 止まってるコンテナ、使われてないボリューム、使われてないネットワーク、使われてないイメージを削除します。
$ docker system prune 
# 個別にやる場合は下記になります。
$ docker image prune
$ docker container prune
$ docker network prune
$ docker volume prune
```

## MySQLのDBとユーザーの作成
```
$ mysql -u root -p --protocol=tcp
```
```
$ CREATE DATABASE laravel;
$ CREATE USER 'laravel'@'%' IDENTIFIED BY 'password';
$ GRANT ALL ON laravel.* TO 'laravel'@'%' WITH GRANT OPTION;
$ FLUSH PRIVILEGES;
$ exit
```


 
