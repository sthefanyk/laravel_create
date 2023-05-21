https://www.digitalocean.com/community/tutorials/how-to-set-up-laravel-nginx-and-mysql-with-docker-compose-on-ubuntu-20-04

https://www.digitalocean.com/community/tutorials/how-to-install-and-set-up-laravel-with-docker-compose-on-ubuntu-22-04#step-1-obtaining-the-demo-application

1. Clone o projeto - git clone https://github.com/laravel/laravel.git projeto
2. > cd projeto
3. > cp .env.example .env

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=nome_database_projeto
DB_USERNAME=sthe
DB_PASSWORD=pass

4. crie Dockerfile

FROM php:7.4-fpm

# Arguments defined in docker-compose.yml
ARG user
ARG uid

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

# Set working directory
WORKDIR /var/www

USER $user

5. > mkdir -p docker-compose/nginx
6. crie arquivo na pasta nginx: projeto.conf

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

7. > mkdir docker-compose/mysql
8. > crie arquivo na pasta mysql: init_db.sql

DROP TABLE IF EXISTS `places`;

CREATE TABLE `places` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `visited` tinyint(1) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=12 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO `places` (name, visited) VALUES ('Berlin',0),('Budapest',0),('Cincinnati',1),('Denver',0),('Helsinki',0),('Lisbon',0),('Moscow',1),('Nairobi',0),('Oslo',1),('Rio',0),('Tokyo',0);

9. crie arquivo na pasta do projeto: docker-compose.yml

version: "3"
services:
  app:
    build:
      args:
        user: sthe
        uid: 1000
      context: ./
      dockerfile: Dockerfile
    image: projeto
    container_name: projeto-app
    restart: unless-stopped
    working_dir: /var/www/
    volumes:
      - ./:/var/www
    networks:
      - projeto

  db:
    image: mysql:8.0
    container_name: projeto-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - ./docker-compose/mysql:/docker-entrypoint-initdb.d
    networks:
      - projeto

  nginx:
    image: nginx:alpine
    container_name: projeto-nginx
    restart: unless-stopped
    ports:
      - 8000:80
    volumes:
      - ./:/var/www
      - ./docker-compose/nginx:/etc/nginx/conf.d/
    networks:
      - projeto

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: projeto-phpmyadmin
    ports:
      - "8012:80"
    depends_on:
      - db
    networks:
      - projeto


networks:
  projeto:
    driver: bridge

10. > docker-compose build app
11. > docker-compose up -d
12. verifica se os containers estão de pé com tudo dentro.

> docker-compose ps
> docker-compose exec app ls -l

13. prepara projeto.

> docker-compose exec app rm -rf vendor composer.lock
> docker-compose exec app composer install
> docker-compose exec app php artisan key:generate

14. bora: http://localhost:8000 

docker-compose exec app php artisan make:controller ClienteController -r
docker-compose exec app php artisan make:model Cliente -m
php artisan make:component Datalist
docker-compose exec app php artisan migrate
docker-compose exec app php artisan migrate:fresh

use App\Models\Cliente;

$table->softDeletes();

use Illuminate\Database\Eloquent\SoftDeletes;
use softDeletes;
protected $fillable = ['nome', 'email'];

