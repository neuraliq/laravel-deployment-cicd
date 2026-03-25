# GitHub Actions Advanced Patterns

## Deploy on Push to Main

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    uses: ./.github/workflows/ci.yml

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/html
            bash deploy.sh
```

## Matrix Testing (Multiple PHP Versions)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        php: ['8.2', '8.3', '8.4']
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
      - run: composer install --no-interaction
      - run: php artisan test
```

## Caching Strategy

```yaml
- name: Cache Composer
  uses: actions/cache@v4
  with:
    path: vendor
    key: composer-${{ hashFiles('composer.lock') }}
    restore-keys: composer-

- name: Cache npm
  uses: actions/cache@v4
  with:
    path: node_modules
    key: npm-${{ hashFiles('package-lock.json') }}
    restore-keys: npm-
```

## Environment File for CI

```env
# .env.ci
APP_ENV=testing
APP_KEY=base64:test-key-here
APP_DEBUG=true
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=testing
DB_USERNAME=root
DB_PASSWORD=password
CACHE_DRIVER=array
QUEUE_CONNECTION=sync
SESSION_DRIVER=array
```
