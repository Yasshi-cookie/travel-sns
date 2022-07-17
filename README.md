# 開発環境構築
## 参考
[【超入門】20分でLaravel開発環境を爆速構築するDockerハンズオン](https://qiita.com/ucan-lab/items/56c9dc3cf2e6762672f4#laravel-%E3%82%A6%E3%82%A7%E3%83%AB%E3%82%AB%E3%83%A0%E7%94%BB%E9%9D%A2%E3%81%AE%E8%A1%A8%E7%A4%BA)

## 手順
```bash
# appコンテナ
$ touch docker-compose.yml
$ mkdir -p infra/php
$ touch infra/php/Dockerfile
$ touch infra/php/php.ini
$ mkdir src

# webコンテナ
$ mkdir infra/nginx
$ touch infra/nginx/default.conf

# Laravelインストール
$ docker compose build
$ docker compose up -d
$ docker compose exec app bash
$ composer create-project --prefer-dist "laravel/laravel=9.*" .
$ chmod -R 777 storage bootstrap/cache
$ php artisan -V
# Laravel Framework 9.1.0
$ exit
# localhost:8081 にアクセスしてLaravelのWelcome画面が表示されればOK
$ docker compose down

# dbコンテナ
$ mkdir infra/mysql
$ touch infra/mysql/Dockerfile
$ touch infra/mysql/my.cnf

# マイグレーション実行
$ docker compose build
$ docker compose up -d
$ docker compose exec app bash
[app] $ php artisan migrate
# Migration table created successfully.
# Migrating: 2014_10_12_000000_create_users_table
# Migrated:  2014_10_12_000000_create_users_table (58.09ms)...
# 上記のように出力されたらOK

# 別ターミナル
$ docker compose exec db bash
[db] $ mysql -u $MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE
[mysql] > show databases;
# +--------------------+
# | Database           |
# +--------------------+
# | information_schema |
# | laravel            |
# +--------------------+
# 2 rows in set (0.00 sec)
```

```yml
docker-compose.yml

version: "3.9"
services:
  app:
    build: ./infra/php
    volumes:
      - ./src:/data

  web:
    image: nginx:1.20-alpine
    ports:
      - 8081:80 # port番号8081は適宜変更してOK
    volumes:
      - ./src:/data
      - ./infra/nginx/default.conf:/etc/nginx/conf.d/default.conf
    working_dir: /data

  db:
    build: ./infra/mysql
    ports:
      - '3306:3306' # 外部からアクセスできるようにport番号を指定する
    volumes:
      - db-store:/var/lib/mysql

volumes:
  db-store:
```

```Dockerfile
infra/php/Dockerfile

FROM php:8.1-fpm-buster

ENV COMPOSER_ALLOW_SUPERUSER=1 \
  COMPOSER_HOME=/composer

COPY --from=composer:2.2 /usr/bin/composer /usr/bin/composer

RUN apt-get update && \
  apt-get -y install --no-install-recommends git unzip libzip-dev libicu-dev libonig-dev && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* && \
  docker-php-ext-install intl pdo_mysql zip bcmath

COPY ./php.ini /usr/local/etc/php/php.ini

WORKDIR /data
```

```ini
infra/php/php.ini

zend.exception_ignore_args = off
expose_php = on
max_execution_time = 30
max_input_vars = 1000
upload_max_filesize = 64M
post_max_size = 128M
memory_limit = 256M
error_reporting = E_ALL
display_errors = on
display_startup_errors = on
log_errors = on
error_log = /dev/stderr
default_charset = UTF-8

[Date]
date.timezone = Asia/Tokyo

[mysqlnd]
mysqlnd.collect_memory_statistics = on

[Assertion]
zend.assertions = 1

[mbstring]
mbstring.language = Japanese
```

```conf
server {
    listen 80;
    server_name example.com;
    root /data/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

```Dockerfile
infra/mysql/Dockerfile

FROM mysql/mysql-server:8.0

ENV MYSQL_DATABASE=travel_sns \
  MYSQL_USER=phper \
  MYSQL_PASSWORD=secret \
  MYSQL_ROOT_PASSWORD=secret \
  TZ=Asia/Tokyo

COPY ./my.cnf /etc/my.cnf
RUN chmod 644 /etc/my.cnf
```

```conf
[mysqld]
# default
skip-host-cache
skip-name-resolve
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
secure-file-priv = /var/lib/mysql-files
user = mysql

pid-file = /var/run/mysqld/mysqld.pid

# character set / collation
character_set_server = utf8mb4
collation_server = utf8mb4_ja_0900_as_cs_ks

# timezone
default-time-zone = SYSTEM
log_timestamps = SYSTEM

# Error Log
log-error = mysql-error.log

# Slow Query Log
slow_query_log = 1
slow_query_log_file = mysql-slow.log
long_query_time = 1.0
log_queries_not_using_indexes = 0

# General Log
general_log = 1
general_log_file = mysql-general.log

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```

```.env
src/.env
src/.env.example

APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:BU4KP9NPIx4bqUHvEDvBnGy/EFLwzB1Y6H+sfzldZFk=
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

# DB接続設定を書き換える
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=travel_sns
DB_USERNAME=phper
DB_PASSWORD=secret
MYSQL_ROOT_PASSWORD=secret

BROADCAST_DRIVER=log
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

MEMCACHED_HOST=127.0.0.1

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=mt1

VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
VITE_PUSHER_PORT="${PUSHER_PORT}"
VITE_PUSHER_SCHEME="${PUSHER_SCHEME}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```
