
# Full Deployment Documentation: Laravel + React (Vite) + Nginx + SSL + MySQL + PHP

## ✅ Server Setup (Ubuntu)

### 1️⃣ Connect to Your Server
```bash
ssh username@your_server_ip
```

### 2️⃣ Update Your Server
```bash
sudo apt update && sudo apt upgrade -y
```

### 3️⃣ Install PHP 8.2 and Required Extensions
```bash
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.2 php8.2-cli php8.2-fpm php8.2-mysql php8.2-curl php8.2-mbstring php8.2-xml php8.2-zip php8.2-bcmath unzip -y
php -v
```

### 4️⃣ Install Composer
```bash
cd ~
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php --version=2.8.9
sudo mv composer.phar /usr/local/bin/composer
composer -v
```

### 5️⃣ Install Node.js and NPM
```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

### 6️⃣ Install MySQL
```bash
sudo apt install mysql-server -y
mysql --version
```

### 7️⃣ MySQL Configuration
```bash
# 📝 Check whether MySQL is running or stopped
sudo systemctl status mysql   

# 📝 Start the MySQL service if it’s not already running
sudo systemctl start mysql    

# 📝 Ensure MySQL will automatically start after server reboot
sudo systemctl enable mysql   

```
##📌📌📌Inside MySQL:
```sql
CREATE USER 'your_username'@'localhost' IDENTIFIED BY 'your_password';
CREATE DATABASE your_database_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON your_database_name.* TO 'your_username'@'localhost';
FLUSH PRIVILEGES;
SHOW DATABASES;
EXIT;
```

### 8️⃣ Install nginx
```bash
# 📝 Install Nginx
sudo apt install nginx -y

# 📝 Enable Nginx to start automatically on system boot
sudo systemctl enable nginx   

# 📝 Start Nginx service immediately
sudo systemctl start nginx     

# 📝 Check Nginx service status
sudo systemctl status nginx 
```

## ✅ Clone & Setup Backend Project
```bash
cd /var/www

# 📝 Clone your repository
sudo git clone your-repo-url

# 📝 Go into the project directory
cd your-project-folder

# 📝 Install dependencies
sudo composer install

# 📝 Copy .env file and generate app key
cp .env.example .env
php artisan key:generate

# 📝 Setup permissions
sudo chown -R www-data:www-data /var/www/your-project-folder
sudo chmod -R 775 /var/www/your-project-folder/storage
sudo chmod -R 775 /var/www/your-project-folder/bootstrap/cache

# ⚠️ Setup your .env file properly with correct DB credentials before running migrations
php artisan migrate

# 📝 Run seeders to insert default data into the database
php artisan db:seed
```





## ✅ Project Clone and Permissions
```bash
cd /var/www

# 📝 Setup backend project
sudo git clone your-repo-url




sudo git clone your-backend-repo-url nobl_backend
sudo git clone your-frontend-repo-url nobl_frontend

sudo chown -R www-data:www-data /var/www/nobl_backend
sudo chmod -R 775 /var/www/nobl_backend/storage
sudo chmod -R 775 /var/www/nobl_backend/bootstrap/cache
```

## ✅ React (Vite) Build Configuration
vite.config.js:
```javascript
export default defineConfig({
  root: ".",
  build: {
    outDir: "../nobl_backend/public",
    emptyOutDir: false,
  },
});
```
```bash
cd /var/www/nobl_frontend
npm install
npm run build
```

## ✅ PHP Configuration for File Upload
```bash
sudo nano /etc/php/8.2/fpm/php.ini
```
Edit:
```
upload_max_filesize = 1024M
post_max_size = 1024M
max_execution_time = 300
memory_limit = 512M
```
```bash
sudo systemctl restart php8.2-fpm
sudo systemctl restart nginx
```

## ✅ SSL (certbot Let's Encrypt)
```bash
sudo systemctl stop nginx
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot certonly --standalone -d noblco.us -d www.noblco.us
sudo systemctl start nginx
```

## ✅ nginx Configuration

### Laravel (API only)
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;
    root /var/www/laravel_project_name/public;
    index index.php index.html;
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### Laravel + React Together
```nginx
server {
    listen 80;
    server_name noblco.us www.noblco.us;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name noblco.us www.noblco.us;
    ssl_certificate /etc/letsencrypt/live/noblco.us/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/noblco.us/privkey.pem;
    client_max_body_size 1024M;
    root /var/www/nobl_backend/public;
    index index.html index.php;
    location / {
        try_files $uri $uri/ /index.html;
    }
    location /api/ {
        try_files $uri $uri/ /index.php?$query_string;
        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.2-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg)$ {
        expires max;
        access_log off;
        add_header Cache-Control "public";
    }
    location ~ /\.ht {
        deny all;
    }
    error_page 404 /index.html;
}
```
```bash
sudo ln -s /etc/nginx/sites-available/noblco /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## ✅ Laravel Queue Worker (Background with tmux)
```bash
cd /var/www/nobl_backend
tmux
php artisan queue:work
```
Detach:
```
Ctrl + B then press D
```
Re-attach:
```bash
tmux attach
```

## ✅ Useful Commands
```bash
# Restart Services
sudo systemctl restart nginx
sudo systemctl restart php8.2-fpm
sudo systemctl restart mysql
# Laravel Permissions (if needed)
sudo chown -R www-data:www-data /var/www/nobl_backend
sudo chmod -R 775 /var/www/nobl_backend/storage
sudo chmod -R 775 /var/www/nobl_backend/bootstrap/cache
```
