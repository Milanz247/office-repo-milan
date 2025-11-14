# WordPress Installation Guide - NGINX, PHP 8.3, MariaDB on CentOS/RHEL 9

This guide provides step-by-step instructions for installing WordPress with NGINX, PHP 8.3, and MariaDB on CentOS/RHEL 9.

## Table of Contents
1. [NGINX Installation and Configuration](#1-nginx-installation-and-configuration)
2. [MariaDB Installation and Setup](#2-mariadb-installation-and-setup)
3. [PHP Installation with Remi Repository](#3-php-installation-with-remi-repository)
4. [PHP-FPM Configuration](#4-php-fpm-configuration)
5. [NGINX Virtual Host Configuration](#5-nginx-virtual-host-configuration)
6. [WordPress Installation and Configuration](#6-wordpress-installation-and-configuration)
7. [SELinux and Permissions](#7-selinux-and-permissions)
8. [Firewall Configuration](#8-firewall-configuration)
9. [Final Checks and Testing](#9-final-checks-and-testing)

---

## 1. NGINX Installation and Configuration

### 1.1 Check Available NGINX Modules
```bash
dnf module list nginx
```

### 1.2 Enable NGINX 1.26 Module
```bash
dnf module enable nginx:1.26
```

### 1.3 Install NGINX
```bash
dnf install nginx -y
```

### 1.4 Enable and Start NGINX Service
```bash
systemctl enable nginx
systemctl start nginx
```

---

## 2. MariaDB Installation and Setup

### 2.1 Install MariaDB Server
```bash
dnf install -y mariadb-server
```

### 2.2 Enable and Start MariaDB Service
```bash
systemctl enable mariadb
systemctl start mariadb
```

### 2.3 Secure MariaDB Installation
Run the security script:
```bash
mysql_secure_installation
```

**Security Configuration Options:**
- Enter current password for root (enter for none): `user@root`
- Switch to unix_socket authentication [Y/n]: `n`
- Change the root password? [Y/n]: `Y` → Set to: `user@root`
- Remove anonymous users: `Y`
- Disallow root login remotely: `Y`
- Remove test database: `Y`
- Reload privilege tables: `Y`

---

## 3. PHP Installation with Remi Repository

### 3.1 Check Available PHP Modules
```bash
dnf module list php
```

### 3.2 Enable PHP 8.3 Module
```bash
dnf module enable php:8.3
```

### 3.3 Install Basic PHP Packages
```bash
dnf install -y php php-fpm php-mysqlnd php-gd php-xml php-mbstring php-json php-intl php-cli
```

### 3.4 Install EPEL Repository (Required for Remi + ImageMagick)
```bash
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

### 3.5 Install Remi Repository (Works Only After EPEL)
```bash
dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
```

### 3.6 Configure Remi Repository for PHP 8.3
```bash
dnf module reset php -y
dnf module enable php:remi-8.3 -y
```

### 3.7 Install ImageMagick and PHP ImageMagick Extension
```bash
dnf install -y ImageMagick ImageMagick-devel
dnf install -y php-pecl-imagick
```

### 3.8 Verify PHP Extensions
```bash
php -m | grep -i imagick
php -m | grep -i gd
```

### 3.9 Enable PHP-FPM Service
```bash
systemctl enable php-fpm
```

---

## 4. PHP-FPM Configuration

### 4.1 Install Text Editor (if needed)
```bash
dnf install nano -y
```

### 4.2 Configure PHP-FPM Pool
Edit the PHP-FPM configuration file:
```bash
nano /etc/php-fpm.d/www.conf
```

**Change these key values in the configuration file:**
```ini
; Run as nginx user
user = nginx
group = nginx

; Listen on Unix socket
listen = /run/php-fpm/www.sock

; Make sure nginx can use the socket
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

### 4.3 Start and Check PHP-FPM Service
```bash
systemctl start php-fpm
systemctl status php-fpm
```

### 4.4 Test NGINX Configuration
```bash
sudo nginx -t
```

---

## 5. NGINX Virtual Host Configuration

### 5.1 Create WordPress Directory
```bash
mkdir -p /var/www/wordpress
chown -R nginx:nginx /var/www/wordpress
```

### 5.2 Create NGINX Virtual Host Configuration
```bash
nano /etc/nginx/conf.d/wordpress.conf
```

**Add the following configuration:**
```nginx
server {
    listen 8081;
    server_name _;  # or wordpress.ceylincolife.internal

    root /var/www/wordpress;
    index index.php index.html index.htm;

    # Log files
    access_log /var/log/nginx/wordpress_access.log;
    error_log  /var/log/nginx/wordpress_error.log;

    # Main location
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP handling
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # Deny access to .ht* files
    location ~ /\.ht {
        deny all;
    }

    # Increase upload size if needed
    client_max_body_size 64M;
}
```

### 5.3 Test and Restart NGINX
```bash
nginx -t
systemctl restart nginx
```

### 5.4 Configure SELinux for Port 8081 (if restart fails)
Allow NGINX to use port 8081:
```bash
sudo semanage port -a -t http_port_t -p tcp 8081
```

If port already exists, modify instead:
```bash
sudo semanage port -m -t http_port_t -p tcp 8081
```

### 5.5 Check NGINX Status
```bash
systemctl restart nginx
systemctl status nginx
```

---

## 6. WordPress Installation and Configuration

### 6.1 Create WordPress Database
Connect to MariaDB:
```bash
sudo mysql -u root -p
```

Execute the following SQL commands:
```sql
CREATE DATABASE wp_ceylinco CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'wp_user@cli!';
GRANT ALL PRIVILEGES ON wp_ceylinco.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 6.2 Transfer WordPress Files
Download WordPress from your PC and transfer it to the server's tmp folder, then:
```bash
cd /tmp
sudo rsync -avP wordpress/ /var/www/wordpress/
```

### 6.3 Set Ownership
```bash
sudo chown -R nginx:nginx /var/www/wordpress
```

### 6.4 Create WordPress Configuration
```bash
cd /var/www/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

**Configure database settings:**
```php
define( 'DB_NAME', 'wp_ceylinco' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'wp_user@cli!' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8mb4' );
define( 'DB_COLLATE', '' );
```

### 6.5 Generate Security Keys
Visit https://api.wordpress.org/secret-key/1.1/salt/ in your browser and copy the generated keys.

**Example keys (replace with your generated ones):**
```php
define('AUTH_KEY',         'lPdExnB= 5j-9?BI-tM}bo36E.nFL@x1+3jf_$q>!*clA#+Za`,)h-urt#]x>-]F');
define('SECURE_AUTH_KEY',  ';j+WKU?u~*2)y/`e#4Dr[/#tvj?Nive|4dt!>+{s6/Cz_{{gnYyqZXd+i=#0 tqC');
define('LOGGED_IN_KEY',    'B`zS,u0;)=iI-8%|<-5A6%j>%dh5F.A+K;U}`wSc+U_ G|g%$F/g3-4=.GY1dV1P');
define('NONCE_KEY',        'dP+`}+%+}HG7s5Lz#CzhO<ptPA1lIoss87<yp!6qK.kAo7/F2~7{9Xj$t&_fX*m6');
define('AUTH_SALT',        '@n7/Phh>nn}e(G+BI8+3hB&Z-3P|K$YP2:uy=|WS|,t[w@!h,63cF:SAfrMLz:=|');
define('SECURE_AUTH_SALT', 'Y[:=(/ndWI$XJ[$[-5P|PpPQNIa>V|}5zM6;Oy=yw+X[|>+L4Sci2bdfoCtc9-R2');
define('LOGGED_IN_SALT',   'zYN8$s/ly1]jZDJ}C|_Hw{Y|}%tWV3H~)D|,L3?s@-})VU*.G1)eMl.=ky{j>AhX');
define('NONCE_SALT',       'rtP{6i pWM7e87`qc^&PiYu3+(BDIiCwuV.q_QfkI.5^]EhdWbuuW;Nl72<tN&LW');
```

**Optional: Add FTP method override:**
```php
define( 'FS_METHOD', 'direct' );
```

---

## 7. SELinux and Permissions

### 7.1 Set File and Directory Permissions
```bash
cd /var/www/wordpress

sudo find . -type d -exec chmod 755 {} \;
sudo find . -type f -exec chmod 644 {} \;

# Tighten wp-config.php permissions
sudo chmod 640 wp-config.php
```

### 7.2 Ensure Proper Ownership
```bash
sudo chown -R nginx:nginx /var/www/wordpress
```

### 7.3 Install SELinux Policy Tools
```bash
sudo dnf install -y policycoreutils-python-utils
```

### 7.4 Configure SELinux Context
```bash
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/wordpress(/.*)?"
sudo restorecon -Rv /var/www/wordpress
```

### 7.5 Allow PHP Web Processes Network Access
```bash
sudo setsebool -P httpd_can_network_connect on
```

### 7.6 Allow NGINX to Use Port 8081
```bash
sudo semanage port -a -t http_port_t -p tcp 8081 2>/dev/null || \
sudo semanage port -m -t http_port_t -p tcp 8081
```

---

## 8. Firewall Configuration

### 8.1 Configure Firewall for Port 8081
If firewalld is running:
```bash
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload
```

**Note:** If you are behind Checkpoint/PAM and there is an additional firewall, ensure port 8081/tcp is allowed from wherever the users are accessing.

---

## 9. Final Checks and Testing

### 9.1 Enable All Services
```bash
sudo systemctl enable nginx php-fpm mariadb
```

### 9.2 Restart All Services
```bash
sudo systemctl restart php-fpm
sudo systemctl restart nginx
```

### 9.3 Check Service Status
```bash
sudo systemctl status nginx
sudo systemctl status php-fpm
sudo systemctl status mariadb
```

### 9.4 Test WordPress Installation
From your jump host or browser, navigate to:
```
http://<server-ip>:8081/
```

### 9.5 Test with cURL Commands
```bash
curl -v http://192.168.57.33:8081
curl -v http://192.168.57.33:8081/wp-admin/install.php
```

---

## Post-Installation Steps

1. Complete the WordPress installation through the web interface
2. Log in to `/wp-admin` to access the WordPress dashboard
3. Configure your WordPress site settings, themes, and plugins as needed

---

## Troubleshooting

- **NGINX fails to start**: Check SELinux port configuration and firewall settings
- **PHP not working**: Verify PHP-FPM service status and socket permissions
- **Database connection issues**: Check MariaDB service status and credentials
- **Permission denied errors**: Review file ownership and SELinux contexts

---

## Security Recommendations

1. Regularly update WordPress, themes, and plugins
2. Use strong passwords for database and WordPress admin accounts
3. Consider implementing SSL/TLS certificates
4. Configure regular backups of your WordPress database and files
5. Monitor log files for suspicious activity

---

*This guide provides a complete WordPress installation with NGINX, PHP 8.3, and MariaDB on CentOS/RHEL 9 systems.*
