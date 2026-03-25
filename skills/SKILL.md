---
name: laravel-deployment-cicd
description: "Laravel deployment pipelines and CI/CD — GitHub Actions, zero-downtime deployment, server provisioning, Docker, Forge, Envoyer, and production checklists. Use when setting up CI/CD pipelines, writing GitHub Actions workflows, configuring deployment scripts, implementing zero-downtime deploys, Dockerizing Laravel apps, setting up staging environments, or preparing for production launch. Triggers on tasks involving deployment automation, CI testing, Docker Compose for Laravel, nginx configuration, Supervisor setup, SSL certificates, or any deployment/DevOps task. Use PROACTIVELY whenever deployment, CI/CD, Docker, production, server setup, or DevOps is mentioned."
compatible_agents:
  - Claude Code
  - Cursor
  - Windsurf
  - Copilot
tags:
  - laravel
  - deployment
  - cicd
  - github-actions
  - docker
  - devops
  - production
  - nginx
  - php
---

# Laravel Deployment & CI/CD

Production deployment pipelines, GitHub Actions CI, Docker setup, and zero-downtime deployment patterns.

## GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  tests:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports: ['3306:3306']
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

      redis:
        image: redis:7
        ports: ['6379:6379']
        options: --health-cmd="redis-cli ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo_mysql, redis
          coverage: xdebug

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}

      - name: Install PHP dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install & build frontend
        run: |
          npm ci
          npm run build

      - name: Prepare environment
        run: |
          cp .env.ci .env
          php artisan key:generate

      - name: Run migrations
        run: php artisan migrate --force
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: password

      - name: Run Pint (code style)
        run: vendor/bin/pint --test

      - name: Run PHPStan
        run: vendor/bin/phpstan analyse

      - name: Run tests
        run: php artisan test --parallel --coverage-clover coverage.xml
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: password
          REDIS_HOST: 127.0.0.1
```

## Zero-Downtime Deployment Script

```bash
#!/bin/bash
# deploy.sh — Generic zero-downtime deployment
set -e

APP_DIR="/var/www/html"
RELEASES_DIR="/var/www/releases"
SHARED_DIR="/var/www/shared"
RELEASE=$(date +%Y%m%d%H%M%S)
NEW_RELEASE="${RELEASES_DIR}/${RELEASE}"

echo "==> Creating release directory..."
mkdir -p "$NEW_RELEASE"

echo "==> Cloning repository..."
git clone --depth 1 --branch main git@github.com:owner/repo.git "$NEW_RELEASE"

echo "==> Linking shared resources..."
ln -sf "${SHARED_DIR}/.env" "${NEW_RELEASE}/.env"
ln -sf "${SHARED_DIR}/storage" "${NEW_RELEASE}/storage"

cd "$NEW_RELEASE"

echo "==> Installing PHP dependencies..."
composer install --no-dev --optimize-autoloader --no-interaction

echo "==> Installing & building frontend..."
npm ci && npm run build

echo "==> Running migrations..."
php artisan migrate --force

echo "==> Caching..."
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

echo "==> Switching symlink (zero-downtime)..."
ln -sfn "$NEW_RELEASE" "$APP_DIR"

echo "==> Reloading services..."
php artisan octane:reload 2>/dev/null || true
php artisan horizon:terminate 2>/dev/null || true
php artisan queue:restart 2>/dev/null || true
sudo systemctl reload php8.3-fpm 2>/dev/null || true

echo "==> Cleaning old releases (keep 5)..."
ls -dt ${RELEASES_DIR}/*/ | tail -n +6 | xargs rm -rf

echo "==> Done! Release: ${RELEASE}"
```

## Docker Setup

```dockerfile
# Dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache \
    nginx supervisor \
    libpng-dev libjpeg-turbo-dev freetype-dev \
    icu-dev libzip-dev oniguruma-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql gd intl zip opcache bcmath pcntl

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html
COPY . .

RUN composer install --no-dev --optimize-autoloader --no-interaction \
    && php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache \
    && chown -R www-data:www-data storage bootstrap/cache

COPY docker/nginx.conf /etc/nginx/http.d/default.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY docker/php.ini /usr/local/etc/php/conf.d/custom.ini

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ['80:80']
    volumes:
      - .env:/var/www/html/.env
      - storage:/var/www/html/storage
    depends_on: [mysql, redis]

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
    volumes: [mysql_data:/var/lib/mysql]

  redis:
    image: redis:7-alpine
    volumes: [redis_data:/data]

volumes:
  mysql_data:
  redis_data:
  storage:
```

## Production Checklist

```
PRE-DEPLOY:
  [ ] APP_ENV=production, APP_DEBUG=false
  [ ] Database migrations tested on staging
  [ ] .env secrets rotated if compromised
  [ ] composer install --no-dev
  [ ] npm run build (assets compiled)
  [ ] php artisan config/route/view:cache

INFRASTRUCTURE:
  [ ] nginx configured with SSL (Let's Encrypt)
  [ ] PHP OPcache enabled (opcache.enable=1)
  [ ] Redis running for cache + sessions + queues
  [ ] Supervisor managing queue workers + Horizon
  [ ] Storage symlink created (php artisan storage:link)
  [ ] File permissions: storage/ and bootstrap/cache/ writable

MONITORING:
  [ ] Error tracking (Sentry / Bugsnag)
  [ ] Laravel Pulse enabled
  [ ] Health check endpoint (/health)
  [ ] Log rotation configured
  [ ] Backup schedule (database + storage)

SECURITY:
  [ ] HTTPS enforced (redirect HTTP → HTTPS)
  [ ] CORS configured for API
  [ ] Rate limiting on auth endpoints
  [ ] Sanctum stateful domains configured
  [ ] Trusted proxies configured (if behind LB)
```

## Key Rules

1. Always run tests in CI before deploy — never deploy untested code
2. Always use zero-downtime deploys (symlink switching) — never deploy in-place
3. Always cache config/routes/views in production — significant performance gain
4. Always use `--no-dev` on `composer install` in production — dev dependencies are a security risk
5. Always run `php artisan migrate --force` in deploy scripts — `--force` required in production
6. Always keep 5 releases for rollback — `ln -sfn releases/previous current` for instant rollback
7. Always use shared storage directory — symlinked across releases for user uploads
8. Always terminate Horizon in deploy — ensures workers pick up new code
9. Always use Supervisor for long-running processes — Horizon, Octane, Reverb, queue workers
10. Always test Docker builds locally before pushing — `docker compose up --build`
