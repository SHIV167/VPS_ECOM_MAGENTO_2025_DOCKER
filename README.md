# Magento 2.4.8 Docker Installation Guide

This guide walks through installing Magento 2.4.8 using Docker. Execute each command one by one.

**Prerequisites**
- Docker & Docker Compose installed
- Domain `shiv.m3.com` pointed to localhost (e.g. via hosts file)

---

## 1. Create `.env` file

```bash
cat > .env <<EOF
MAGENTO_HOST=shiv.m3.com
MYSQL_ROOT_PASSWORD=root
MAGENTO_DB_NAME=magento
MAGENTO_DB_USER=magento
MAGENTO_DB_PASSWORD=magento
REDIS_HOST=redis
REDIS_PORT=6379
OPENSEARCH_HOST=opensearch
OPENSEARCH_PORT=9200
EOF
```

## 2. Create `docker-compose.yml`

```yaml
version: '3.7'
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MAGENTO_DB_NAME}
      MYSQL_USER: ${MAGENTO_DB_USER}
      MYSQL_PASSWORD: ${MAGENTO_DB_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    ports:
      - "3306:3306"

  redis:
    image: redis:6.2
    ports:
      - "${REDIS_PORT}:6379"

  opensearch:
    image: opensearchproject/opensearch:1.2.4
    environment:
      - discovery.type=single-node
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - opensearch-data:/usr/share/opensearch/data
    ports:
      - "${OPENSEARCH_PORT}:9200"

  php:
    build:
      context: .
      dockerfile: php/Dockerfile
    volumes:
      - ./magento:/var/www/html
    environment:
      COMPOSER_HOME: /tmp
    depends_on:
      - db
      - redis
      - opensearch
    ports:
      - "9000:9000"

  web:
    image: nginx:1.21-alpine
    depends_on:
      - php
    volumes:
      - ./magento:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"

volumes:
  db-data:
  opensearch-data:
```

## 3. Create `php/Dockerfile`

```dockerfile
FROM php:7.4-fpm

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install PHP extensions
RUN apt-get update && apt-get install -y \
    libfreetype6-dev libjpeg62-turbo-dev libpng-dev libxml2-dev libonig-dev libzip-dev zip unzip git \
  && docker-php-ext-configure gd --with-freetype --with-jpeg \
  && docker-php-ext-install -j$(nproc) bcmath gd intl mbstring mysqli pdo_mysql soap xsl zip opcache \
  && pecl install redis \
  && docker-php-ext-enable redis
```

## 4. Create Nginx config `nginx/default.conf`

```nginx
server {
    listen 80;
    server_name ${MAGENTO_HOST};
    set $MAGE_ROOT /var/www/html;
    include /var/www/html/nginx.conf.sample;
}
```

## 5. Start containers

```bash
docker-compose up -d
```

## 6. Download Magento

```bash
docker exec -it php bash -c "composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition=2.4.8 /var/www/html"
```

## 7. Install Magento

```bash
docker exec -it php bash -c "cd /var/www/html && bin/magento setup:install \
  --base-url=http://${MAGENTO_HOST} \
  --db-host=db \
  --db-name=${MAGENTO_DB_NAME} \
  --db-user=${MAGENTO_DB_USER} \
  --db-password=${MAGENTO_DB_PASSWORD} \
  --admin-firstname=Admin \
  --admin-lastname=User \
  --admin-email=admin@shiv.m3.com \
  --admin-user=admin \
  --admin-password=Admin123! \
  --language=en_US \
  --currency=USD \
  --timezone=UTC \
  --use-rewrites=1"
```

## 8. File permissions & deployment

```bash
docker exec -it php bash -c "cd /var/www/html && \
  find var generated pub/static pub/media app/etc -type f -exec chmod g+w {} + && \
  find var generated pub/static pub/media app/etc -type d -exec chmod g+ws {} + && \
  chown -R www-data:www-data ."
```

```bash
docker exec -it php bash -c "cd /var/www/html && \
  bin/magento deploy:mode:set developer && \
  bin/magento setup:upgrade && \
  bin/magento setup:di:compile && \
  bin/magento indexer:reindex && \
  bin/magento setup:static-content:deploy -f && \
  bin/magento cache:flush"
```

## 9. Sample data

```bash
docker exec -it php bash -c "cd /var/www/html && \
  composer require magento/sample-data && \
  bin/magento sampledata:deploy && \
  bin/magento setup:upgrade && \
  bin/magento cache:flush"
```

## 10. Configure Redis caching

Add the following to `app/etc/env.php` inside the returned array:

```php
'session' => [
    'save' => 'redis',
    'redis' => [
        'host' => '${REDIS_HOST}',
        'port' => '${REDIS_PORT}',
        'database' => 2
    ]
],
'cache' => [
    'frontend' => [
        'default' => [
            'backend' => 'Cm_Cache_Backend_Redis',
            'backend_options' => [
                'server' => '${REDIS_HOST}',
                'port' => '${REDIS_PORT}',
                'database' => 0
            ]
        ],
        'page_cache' => [
            'backend' => 'Cm_Cache_Backend_Redis',
            'backend_options' => [
                'server' => '${REDIS_HOST}',
                'port' => '${REDIS_PORT}',
                'database' => 1
            ]
        ]
    ]
],
```  

Finally, restart PHP:

```bash
docker-compose restart php
```

Your Magento 2.4.8 instance is now running at http://${MAGENTO_HOST}. Access the admin panel at `http://${MAGENTO_HOST}/admin`.
