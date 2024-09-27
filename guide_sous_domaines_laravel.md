
# Guide for Configuring Custom and Dynamic Subdomains for Laravel

This guide provides detailed instructions on configuring custom subdomains in a development environment, using subdomains in Laravel, and managing dynamic subdomains in production with cPanel.

---

## a. Configure the Development Environment to Accept Custom Subdomains

### Windows:
1. Open the file `C:\Windows\System32\drivers\etc\hosts`.
2. Add a line for each custom subdomain:
   ```
   127.0.0.1 subdomain.localhost
   ```
   
### Mac OS / Linux:
1. Edit the `/etc/hosts` file with administrator privileges:
   ```
   sudo nano /etc/hosts
   ```
2. Add a similar line:
   ```
   127.0.0.1 subdomain.localhost
   ```

---

## b. Configure VirtualHost on Nginx and Apache to Manage Subdomains

### Apache (Windows, Mac OS, Linux):
1. Enable the required Apache modules:
   ```
   sudo a2enmod rewrite
   sudo systemctl restart apache2
   ```
2. Create a VirtualHost for the subdomains:
   ```apache
   <VirtualHost *:80>
       ServerAlias *.localhost
       DocumentRoot /path/to/laravel/public
       RewriteEngine On
       RewriteCond %{HTTP_HOST} ^(?<subdomain>\w+)\.localhost$
       RewriteRule ^ - [E=SUBDOMAIN:%{subdomain}]
   </VirtualHost>
   ```

### Nginx (Windows, Mac OS, Linux):
1. Create an Nginx server block to capture subdomains:
   ```nginx
   server {
       listen 80;
       server_name ~^(?<subdomain>\w+)\.localhost$;
       root /path/to/laravel/public;

       location / {
           try_files $uri $uri/ /index.php?$query_string;
       }

       location ~ \.php$ {
           include snippets/fastcgi-php.conf;
           fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
       }
   }
   ```

2. Restart Nginx:
   ```
   sudo systemctl restart nginx
   ```

---

## c. Using Subdomains in Laravel

### Step 1: Create Middleware to Handle Subdomains
Run the command to generate middleware:
```
php artisan make:middleware SubdomainMiddleware
```

Add the following code to dynamically configure the database based on the subdomain:
```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\DB;

class SubdomainMiddleware
{
    public function handle($request, Closure $next)
    {
        $host = $request->getHost();
        $pattern = '/(?<subdomain>\w+)\.localhost/';

        if (preg_match($pattern, $host, $matches)) {
            $subdomain = $matches['subdomain'];
            dd($subdomain);
        }

        return $next($request);
    }
}
```

### Step 2: Register the Middleware
In `app/Http/Kernel.php`, add the middleware:
```php
protected $middlewareGroups = [
    'web' => [
        // other middlewares
        \App\Http\Middleware\SubdomainMiddleware::class,
    ],
];
```

---

## d. Managing Dynamic Subdomains with cPanel

### Step 1: Create a Wildcard Subdomain
1. Log in to cPanel.
2. Go to **Domains** > **Subdomains**.
3. Create a subdomain with `*` as the subdomain name.

### Step 2: Configure a Wildcard SSL Certificate
1. Go to **SSL/TLS** > **Manage SSL**.
2. Use **AutoSSL** or manually install a wildcard SSL certificate with Let's Encrypt.
   
### Step 3: Use the Middleware in Laravel
The previously defined middleware will work in production for dynamic subdomains like `subdomain.yourdomain.com`.

### Step 4: Manage Redirections on cPanel
- Go to **Domains** > **Redirects** to create specific redirects.
- Or directly modify the `.htaccess` file to force HTTPS or redirect subdomains.

```apache
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteCond %{HTTP_HOST} ^(.*)\.yourdomain\.com$ [NC]
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

---

### Additional References:
- [Laravel Middleware Documentation](https://laravel.com/docs/8.x/middleware)
- [Nginx Configuration Documentation](https://nginx.org/en/docs/)
- [Apache VirtualHost Documentation](https://httpd.apache.org/docs/2.4/vhosts/)
- [cPanel Documentation](https://docs.cpanel.net/)