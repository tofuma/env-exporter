# ENV Exporter

A simple PHP package to generate `.env` files from example templates using system environment variables.

Created by [tofuma](https://github.com/tofuma).

## What does this package do?

When you deploy an application, you usually have a `.env.example` file with all the environment variable keys your application needs. The problem is: how do you generate the actual `.env` file with the correct values from your server environment?

This package solves that problem. It reads the keys from your `.env.example` file and creates a new `.env` file containing only the variables that exist in your system environment.

### Why is this useful?

- **Security**: Only exports variables that your application actually needs (defined in `.env.example`)
- **Automation**: No manual copy-paste of environment variables during deployment
- **Consistency**: Ensures your `.env` file always matches your `.env.example` structure

## Requirements

- PHP 7.1 or higher

## How it works

1. The package reads your `.env.example` file
2. It extracts all the variable keys (for example: `APP_NAME`, `DB_HOST`, `REDIS_URL`)
3. For each key, it checks if that variable exists in the system environment
4. If the variable exists, it adds the key and value to the output
5. It writes the result to your `.env` file

### Supported formats in .env.example

The package understands these formats:

```bash
# Simple key=value
APP_NAME=Laravel

# Empty value
APP_ENV=

# Quoted values
APP_NAME="My App"

# With export prefix
export APP_NAME=Laravel

# Comments are ignored
# This is a comment
; This is also a comment
```

## Installation

```bash
composer require tofuma/env-exporter
```

## Usage

### Option 1: Using the static method in PHP code

```php
<?php

use Tofuma\EnvExporter\EnvExporter;

// Generate .env file from .env.example
EnvExporter::generate('.env.example', '.env');

// Or get the key-value pairs as an array (without writing to file)
$pairs = EnvExporter::generate('.env.example', '.env', true);
print_r($pairs);
// Output: ['APP_NAME' => 'MyApp', 'DB_HOST' => 'localhost', ...]

// Generate file and get the count of exported variables
$count = EnvExporter::generateAndReport('.env.example', '.env');
echo "Exported {$count} variables";
```

### Option 2: Using the command line interface (CLI)

```bash
vendor/bin/env-exporter .env.example .env
```

Output:
```
Generated .env with 15 entries: .env
```

## Real World Examples

### Example 1: Simple deployment script

```bash
#!/bin/bash

# Your deployment script
cd /var/www/myapp
git pull origin main
composer install --no-dev

# Generate .env from system environment variables
vendor/bin/env-exporter .env.example .env

php artisan migrate --force
```

### Example 2: Docker entrypoint script

In your `docker-entrypoint.sh`:

```bash
#!/bin/bash

# Generate .env file from container environment variables
vendor/bin/env-exporter .env.example .env

# Start the application
php-fpm
```

In your `Dockerfile`:

```dockerfile
FROM php:8.2-fpm

WORKDIR /var/www/html

COPY . .
RUN composer install --no-dev

COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]
```

In your `docker-compose.yml`:

```yaml
services:
  app:
    build: .
    environment:
      APP_NAME: "My Application"
      APP_ENV: production
      APP_DEBUG: "false"
      DB_HOST: database
      DB_DATABASE: myapp
      DB_USERNAME: root
      DB_PASSWORD: secret
```

### Example 3: Docker with run.sh script

Many projects use a `run.sh` script to start the application inside a Docker container. This is a common pattern for applications that need to do some setup before starting.

In your `run.sh`:

```bash
#!/bin/bash
set -e

# Generate .env file from container environment variables
vendor/bin/env-exporter .env.example .env

# Run database migrations (optional)
php artisan migrate --force

# Clear and cache configurations (optional)
php artisan config:cache
php artisan route:cache

# Start the application
exec php-fpm
```

In your `Dockerfile`:

```dockerfile
FROM php:8.2-fpm

WORKDIR /var/www/html

# Install dependencies
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts

# Copy application code
COPY . .

# Make run.sh executable
RUN chmod +x run.sh

# Use run.sh as the entrypoint
CMD ["./run.sh"]
```

In your `docker-compose.yml`:

```yaml
services:
  app:
    build: .
    environment:
      APP_NAME: "My Application"
      APP_ENV: production
      APP_DEBUG: "false"
      APP_KEY: base64:your-app-key-here
      APP_URL: https://myapp.com
      
      DB_CONNECTION: mysql
      DB_HOST: database
      DB_PORT: 3306
      DB_DATABASE: myapp
      DB_USERNAME: root
      DB_PASSWORD: secret
      
      CACHE_DRIVER: redis
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      - database
      - redis

  database:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: myapp
      MYSQL_ROOT_PASSWORD: secret

  redis:
    image: redis:alpine
```

### Example 4: Using with Dokploy

[Dokploy](https://dokploy.com) is a deployment platform that allows you to set environment variables through its web interface. When you add environment variables in Dokploy, they become available as system environment variables in your container.

**The problem**: Dokploy (and similar platforms) may inject many system variables that you do not need. If you try to export all environment variables, you will get many unnecessary variables.

**The solution**: This package reads only the keys defined in your `.env.example` file. This means you only get the variables your application needs.

**Step 1**: In your Dokploy project, go to the "Environment" tab and add your variables:

```
APP_NAME=My Application
APP_ENV=production
APP_DEBUG=false
APP_URL=https://myapp.com

DB_CONNECTION=mysql
DB_HOST=your-database-host
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=myuser
DB_PASSWORD=mysecretpassword

REDIS_HOST=your-redis-host
REDIS_PORT=6379
```

**Step 2**: In your project, make sure you have a `.env.example` file with all the keys:

```
APP_NAME=
APP_ENV=
APP_DEBUG=
APP_URL=

DB_CONNECTION=
DB_HOST=
DB_PORT=
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=

REDIS_HOST=
REDIS_PORT=
```

**Step 3**: Create a `run.sh` script in your project root:

```bash
#!/bin/bash
set -e

# Generate .env file from Dokploy environment variables
vendor/bin/env-exporter .env.example .env

# Start your application
exec php-fpm
```

**Step 4**: In your Dockerfile, use the `run.sh` script:

```dockerfile
FROM php:8.2-fpm

WORKDIR /var/www/html

COPY . .
RUN composer install --no-dev
RUN chmod +x run.sh

CMD ["./run.sh"]
```

The package will create a `.env` file with only the variables defined in `.env.example`, using the values from Dokploy environment settings.

### Example 5: Kubernetes deployment

In your Kubernetes deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          envFrom:
            - secretRef:
                name: myapp-secrets
            - configMapRef:
                name: myapp-config
```

In your container `run.sh`:

```bash
#!/bin/bash
set -e

vendor/bin/env-exporter .env.example .env
exec php-fpm
```

### Example 6: GitHub Actions deployment

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install dependencies
        run: composer install --no-dev
      
      - name: Generate .env file
        run: vendor/bin/env-exporter .env.example .env
        env:
          APP_NAME: ${{ vars.APP_NAME }}
          APP_ENV: production
          DB_HOST: ${{ secrets.DB_HOST }}
          DB_DATABASE: ${{ secrets.DB_DATABASE }}
          DB_USERNAME: ${{ secrets.DB_USERNAME }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      
      - name: Deploy
        run: ./deploy.sh
```

### Example 7: Laravel Forge / Envoyer

In your deployment script:

```bash
cd /home/forge/myapp.com

# Generate .env from server environment variables
vendor/bin/env-exporter .env.example .env

php artisan config:cache
php artisan route:cache
php artisan view:cache
```

## License

MIT License. See [LICENSE](LICENSE) for more information.

## Author

Created and maintained by [tofuma](https://github.com/tofuma).
