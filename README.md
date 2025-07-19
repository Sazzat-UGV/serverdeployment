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

## ✅ Setting Up MySQL Database and User

```bash
# 📝  Use sudo to login to MySQL as root without password
sudo mysql

# 📝 Create a new MySQL user with password
CREATE USER 'your_username'@'localhost' IDENTIFIED BY 'your_password';

# 📝 Create a new database with UTF8MB4 charset for full Unicode support
CREATE DATABASE your_database_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# 📝 Grant all privileges on the new database to the new user
GRANT ALL PRIVILEGES ON your_database_name.* TO 'your_username'@'localhost';

# 📝 Reload the privilege tables to apply changes
FLUSH PRIVILEGES;

# 📝 Exit MySQL prompt
EXIT;

# 📝 Login as the new user to verify credentials and access
mysql -u your_username -p

```

```bash
# 📌 MySQL commands for database and table management

# 📝 Show all databases
SHOW DATABASES;

# 📝 Select a database to work with
USE your_database_name;

# 📝 Show all tables in the selected database
SHOW TABLES;

# 📝 Describe the structure of a table
DESCRIBE table_name;

# 📝 Delete (drop) a table permanently
DROP TABLE table_name;

# 📝 Delete (drop) a database permanently
DROP DATABASE your_database_name;

# 📝 Exit the MySQL client
EXIT;

# ⚠️ WARNING: DROP commands permanently delete data and cannot be undone!
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

# 📝 Run seeders to insert default data into the database (optional)
php artisan db:seed

# 📝 Generate JWT secret key (optional, if your app uses JWT authentication)
php artisan jwt:secret
```

## ✅ Clone & Setup Frontend Project

```bash
cd /var/www

# 📝 Clone your repository
sudo git clone your-repo-url

# 📝 Go into the project directory
cd your-project-folder
```

```javascript
# 📝 Vite Build Configuration (vite.config.js)
export default defineConfig({
  root: ".",
  build: {
    outDir: "../your_backend_project/public",
    emptyOutDir: false,
  },
});

```

```bash
# 📌 Vite commands

# 📝 Install dependencies
sudo npm install

# 📝 Build production assets (Setup Backend Domain / IP for API Calls (Required Before Build))
sudo npm run build
```

## ✅ SSL Setup with Certbot

```bash
# 🛑 Stop Nginx to free up port 80 for certbot standalone verification
sudo systemctl stop nginx

# 📥 Install certbot via snap
sudo snap install --classic certbot

# 🔗 Create a symbolic link for easy access to certbot command
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# 🚀 Obtain SSL certificate for your domain (replace with your actual domain)
sudo certbot certonly --standalone -d example.com -d www.example.com

# ▶️ Restart Nginx after successfully issuing the SSL certificate
sudo systemctl start nginx
```

## ✅ Configure Nginx

### 1️⃣ Go to the Available Sites Directory
```bash
cd /etc/nginx/sites-available/
```

### 2️⃣ Create a New Config File
```bash
sudo nano /etc/nginx/sites-available/your_conf_file_name.conf
```

### 3️⃣ Paste Your Nginx Configuration
### Laravel (API only)

```nginx
server {
    listen 80;
    server_name your_domain;   # Replace with your actual domain (e.g., yourdomain.com www.yourdomain.com)

    root /var/www/backend_project_repo/public;  # Path to Laravel public directory
    index index.php index.html;

    # 🔧 Allow large file uploads (adjust as needed)
    client_max_body_size 1024M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;  # Adjust PHP version if needed
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### Laravel + React Together (With SSL)

```nginx
server {
    listen 80;
    server_name your_domain;   # Replace with your actual domain (e.g., yourdomain.com www.yourdomain.com)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name your_domain;   # Replace with your actual domain (e.g., yourdomain.com www.yourdomain.com)
   
    # SSL Certificates 
    ssl_certificate /etc/letsencrypt/live/noblco.us/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/noblco.us/privkey.pem;

    # Allow large file uploads (adjust as needed)
    client_max_body_size 1024M;

    # Project root (Laravel public directory)
    root /var/www/backend_project_repo/public;
    index index.html index.php;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        try_files $uri $uri/ /index.php?$query_string;
        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.2-fpm.sock;   # Adjust PHP version if needed
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;  # Adjust PHP version if needed
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

### Laravel + React Together (Without SSL)

```nginx
server {
    listen 80;
    server_name your_domain;  # Replace with your actual domain (e.g., yourdomain.com www.yourdomain.com)

    # Laravel public directory
    root /var/www/backend_project_repo/public;
    index index.html index.php;

    # Allow large file uploads
    client_max_body_size 1024M;

    # Frontend React SPA support
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Laravel API routes
    location /api/ {
        try_files $uri $uri/ /index.php?$query_string;

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.2-fpm.sock;  # Adjust PHP version if needed
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;  # Adjust PHP version if needed
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
### 4️⃣ Enable the Site
```bash
# 📝 Create a symbolic link
sudo ln -s /etc/nginx/sites-available/your_conf_file_name.conf /etc/nginx/sites-enabled/
```

### 5️⃣ Test Nginx Config
```bash
sudo nginx -t
```

### 6️⃣ Reload Nginx
```bash
sudo systemctl reload nginx
```

## ✅ PHP Configuration for File Upload

```bash
sudo nano /etc/php/8.2/fpm/php.ini
```
```
# 📝 Find and edit
upload_max_filesize = 1024M
post_max_size = 1024M
max_execution_time = 300
memory_limit = 512M
```

```bash
# 📝 Restart PHP
sudo systemctl restart php8.2-fpm

# 📝 Restart Nginx
sudo systemctl restart nginx
```
## ✅ tmux Command Guide

```bash
# 📝 New default session start
tmux 

# 📝 Detach from the current tmux session without stopping it
#    Press: Ctrl + B, then press D

# 📝 Kill/terminate a specific tmux session
tmux kill-session -t session_name

# 📝 List all active tmux sessions
tmux ls

# 📝 Kill/terminate all tmux sessions
tmux kill-server
```

## ✅ Useful Commands

```bash
# 📝 Restart Nginx web server
sudo systemctl restart nginx       

# 📝 Restart PHP-FPM service (adjust PHP version if needed)
sudo systemctl restart php8.2-fpm  

# 📝 Restart MySQL database server
sudo systemctl restart mysql       

```
