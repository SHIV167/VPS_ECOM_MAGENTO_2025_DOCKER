FROM php:8.2-fpm

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install PHP extensions
RUN apt-get update && apt-get install -y \
    libfreetype6-dev libjpeg62-turbo-dev libpng-dev libxml2-dev libonig-dev libzip-dev zip unzip git libxslt1-dev autoconf build-essential \
  && docker-php-ext-configure gd --with-freetype --with-jpeg \
  && docker-php-ext-install -j$(nproc) bcmath gd intl mbstring mysqli pdo_mysql soap xsl zip opcache ftp \
  && pecl install redis \
  && docker-php-ext-enable redis
