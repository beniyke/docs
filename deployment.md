# Deployment

Deploying an Anchor application involves a few common steps to ensure your application runs securely and efficiently in a production environment.

## Server Requirements

Ensure your server meets the following requirements:

- **PHP**: >= 8.2
- **Extensions**:
  - BCMath
  - Ctype
  - JSON
  - Mbstring
  - OpenSSL
  - PDO
  - Tokenizer
  - XML
  - cURL

## Installation

1.  **Clone the Repository**:

    ```bash
    git clone https://github.com/beniyke/anchor.git /var/www/your-app
    ```

2.  **Install Dependencies**:

    ```bash
    cd /var/www/your-app
    composer install --optimize-autoloader --no-dev
    ```

3.  **Install System Packages**:
    Ensure the queue system package is installed for production.
    ```bash
    php dock package:install Queue --system --force
    ```

## Configuration

1.  **Environment Variables**:
    Copy the `.env.example` file to `.env` and configure your production environment variables.

    ```bash
    cp .env.example .env
    nano .env
    ```

    Ensure `APP_ENV` is set to `production` and `APP_DEBUG` is set to `false`.

2.  **Generate Application Key**:
    Generate a new application key to ensure session and encryption security.
    ```bash
    php dock key:generate
    ```

## Directory Permissions

Ensure the web server user (e.g., `www-data`) has write permissions to the `storage` directory and any other directories where the application needs to write files.

```bash
chown -R www-data:www-data /var/www/your-app
chmod -R 775 /var/www/your-app/storage
```

## Web Server Configuration

### Nginx

Here is an example Nginx configuration for an Anchor application:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/your-app;

    add_header X-Frame-Options "SAMEORIGIN";
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
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### Apache

If you are using Apache, ensure `mod_rewrite` is enabled. You can use the following `.htaccess` file in your `public` directory:

```apache
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```

## Optimization

1.  **Autoloader Optimization**:
    The `composer install` command with `--optimize-autoloader` already handles this.

2.  **Clear Cache**:
    Clear any existing application cache.
    ```bash
    php dock cache:flush
    ```

## Database Migrations

Run your database migrations to ensure the production database schema is up to date.

```bash
php dock migration:run
```

## Queues

If your application uses queues, you will need to configure a process monitor like Supervisor to keep your queue workers running.

Example Supervisor configuration:

```ini
[program:anchor-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/your-app/dock worker:start --queue=default --memory=128
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/your-app/storage/logs/worker.log
```
