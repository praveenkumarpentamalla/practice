Here are **Dockerfiles rewritten using only the required keywords**:

* `FROM`
* `COPY`
* `RUN`
* `CMD`
* `ENTRYPOINT`
* `WORKDIR`
* `EXPOSE`
* `VOLUME`

---

## 🟩 **Node.js Dockerfile**

```dockerfile
# FROM: Use official Node.js image
FROM node:18-slim

# WORKDIR: Set working directory
WORKDIR /usr/src/app

# COPY: Copy package files
COPY package*.json ./

# RUN: Install dependencies
RUN npm install

# COPY: Copy application code
COPY . .

# VOLUME: Mount volume for logs
VOLUME ["/usr/src/app/logs"]

# EXPOSE: Port app will run on
EXPOSE 3000

# ENTRYPOINT: Base command
ENTRYPOINT ["node"]

# CMD: App entry file
CMD ["server.js"]
```

---

## 🟦 **PHP Dockerfile (Plain PHP App)**

```dockerfile
# FROM: Use official PHP Apache image
FROM php:8.2-apache

# WORKDIR: Set working directory
WORKDIR /var/www/html

# COPY: Copy all source files
COPY . .

# RUN: Enable Apache mod_rewrite
RUN a2enmod rewrite

# VOLUME: Optional for file uploads or persistent logs
VOLUME ["/var/www/html/uploads"]

# EXPOSE: Apache listens on port 80
EXPOSE 80

# ENTRYPOINT: Apache runs by default, override if needed
ENTRYPOINT ["apache2-foreground"]

# CMD: Not needed explicitly as apache2-foreground is ENTRYPOINT
CMD []
```

---

## 🟥 **Laravel Dockerfile**

```dockerfile
# FROM: Use PHP with FPM (for Laravel apps)
FROM php:8.2-fpm

# WORKDIR: Laravel root folder
WORKDIR /var/www

# COPY: Copy application code
COPY . .

# RUN: Install dependencies
RUN apt-get update && apt-get install -y \
    git zip unzip libzip-dev libpng-dev \
    && docker-php-ext-install pdo pdo_mysql zip

# COPY: Install Composer
COPY --from=composer:2.6 /usr/bin/composer /usr/bin/composer

# RUN: Install Laravel packages
RUN composer install --no-dev --optimize-autoloader

# VOLUME: For storage and logs
VOLUME ["/var/www/storage", "/var/www/bootstrap/cache"]

# EXPOSE: Laravel with PHP-FPM runs on port 9000
EXPOSE 9000

# ENTRYPOINT: Start PHP-FPM
ENTRYPOINT ["php-fpm"]

# CMD: Optional if php-fpm is default
CMD []
```

---

### ✅ Summary

| Stack   | Port | VOLUME                                 | ENTRYPOINT           |
| ------- | ---- | -------------------------------------- | -------------------- |
| Node.js | 3000 | `/usr/src/app/logs`                    | `node`               |
| PHP     | 80   | `/var/www/html/uploads`                | `apache2-foreground` |
| Laravel | 9000 | `/var/www/storage`, `/bootstrap/cache` | `php-fpm`            |

Let me know if you want Docker Compose for any of these!
