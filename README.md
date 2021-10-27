# deployment-guide-laravel

### Getting Started (AWS EC2 instance Ubuntu 20.04)

Download your AWS key, locate it in any folder and run:

```
- chmod 400 Your-Key.pem
- ssh ubuntu@your-ec2-ip // run command from directory where Your-Key.pem stored
```

```
- Change password on first login (not in case of AWS EC2)
- adduser laravel
- Enter password and other information
- usermod -aG sudo laravel
```

### Locking Down to SSH Key only (Extremely Important)

```
- In your local machine, ssh-keygen
- Generate a key, if you leave passphrase blank, no need for password
- ls ~/.ssh to show files in your local machine
- Get the public key, cat ~/.ssh/id_rsa.pub
- Copy it
- su laravel then mkdir ~/.ssh fix permissions chmod 700 ~/.ssh in your production server
- nano ~/.ssh/authorized_keys and paste key
- chmod 600 ~/.ssh/authorized_keys to restrict this from being modified
- exit to return to root user
```

### Disable Password from Server

```
- sudo nano /etc/ssh/sshd_config
- Find PasswordAuthentication and set that to no
- Turn on PubkeyAuthentication yes
- Turn off ChallengeResponseAuthentication no
- Reload the SSH service sudo systemctl reload sshd
- Test new user in a new tab to prevent getting locked out
```

### Setting Up Firewall

```
- View all available firewall settings
- sudo ufw app list
- Allow on OpenSSH so we don't get locked out
- sudo ufw allow OpenSSH
- Enable Firewall
- sudo ufw enable
- Check the status
- sudo ufw status
```

### Install Nginx, MySQL, PHP

#### Nginx

```
- sudo apt update enter root password
- sudo apt install nginx enter Y to install
- sudo ufw app list For firewall
- sudo ufw allow 'Nginx HTTP' to add NGINX
- sudo ufw status to verify change
- Visit server in browser
```

#### MySQL 8 (in Ubuntu 20.04 this version will be downloaded by default, one need 1Gb RAM min)

```
- sudo apt install mysql-server // enter Y to install
- sudo mysql_secure_installation // to run automated securing script
- Press N for VALIDATE PASSWORD plugin
- Set root password
- Remove anonymous users? Y
- Disallow root login remotely? N
- Remove test database and access to it? Y
- Reload privilege tables now? Y
- sudo mysql to enter MySQL CLI
- SELECT user,authentication_string,plugin,host FROM mysql.user; // to verify root user's auth method
- ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'STRONG_PASSWORD_HERE'; // to set a root password
- SELECT user,authentication_string,plugin,host FROM mysql.user; // to verify root user's auth method
- FLUSH PRIVILEGES; // to apply all changes
- mysql -u root -p to access db from now on, enter password STRONG_PASSWORD_HERE
```

#### PHP & Basic Nginx

```
- sudo add-apt-repository universe to add software repo
- sudo apt install software-properties-common
- sudo add-apt-repository ppa:ondrej/php // to instal PHP 8
- sudo apt install curl
- sudo apt install php8.0-fpm php8.0-mysql php8.0-gd php8.0-mbstring php8.0-bcmath php8.0-xml php8.0-zip php8.0-curl// to install the basic PHP software
- sudo systemctl status php8.0-fpm // to check status
- sudo nano /etc/nginx/sites-available/YOUR.DOMAIN.COM (if you haven't bought domain specify any)
```

```
server {
        listen 80;
        root /var/www/html;
        index index.php index.html index.htm index.nginx-debian.html;
        server_name YOUR.DOMAIN.COM;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        }

        location ~ /\.ht {
                deny all;
        }
}

```

```
- sudo ln -s /etc/nginx/sites-available/YOUR.DOMAIN.COM /etc/nginx/sites-enabled/ to create symlink to enabled sites
- sudo unlink /etc/nginx/sites-enabled/default to remove default link
- sudo nginx -t test the whole config
- sudo systemctl reload nginx to apply all changes
- sudo nano /var/www/html/info.php to start a new PHP file, fill it with <?php phpinfo();
- sudo rm /var/www/html/info.php optional command to get rid of test file
```

### Let's Dial in The Laravel Ecosystem

```
- sudo apt-get install php8.0-mbstring php8.0-xml composer unzip // composer should be updated after to version 2
- mysql -u root -p Login to create the Laravel DB
- CREATE USER 'laraveluser'@'localhost' IDENTIFIED BY 'your password here';
- CREATE DATABASE laravel DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
- GRANT ALL ON laravel.* TO 'laraveluser'@'localhost';
- FLUSH PRIVILEGES;
- exit
- cd /var/www/html, sudo mkdir -p project-name
- sudo chown laravel:laravel project-name
- git clone https://github.com/your-repo/laravel-project.git .
- composer install
- cp .env.example .env, and then nano .env
```

- APP_NAME=Laravel
- APP_ENV=production
- APP_KEY=
- APP_DEBUG=false
- APP_URL=http://YOUR.DOMAIN.COM

- LOG_CHANNEL=stack

- DB_CONNECTION=mysql
- DB_HOST=127.0.0.1
- DB_PORT=3306
- DB_DATABASE=laravel
- DB_USERNAME=laraveluser
- DB_PASSWORD=STRONG_PASSWORD_HERE

```
- php artisan migrate
- php artisan key:generate // to generate the key
- sudo chgrp -R www-data storage bootstrap/cache // to fix permissions
- sudo chmod -R ug+rwx storage bootstrap/cache // to fix permissions
- sudo chmod -R 755 /var/www/html/project-name // to fix permissions
- chmod -R 777 /var/www/html/project-name/storage // to fix permission
- php artisan storage:link
```

### Modify Nginx

```
- sudo nano /etc/nginx/sites-available/YOUR.DOMAIN.COM
```

```
server {
    listen 80;
    listen [::]:80;

    root /var/www/html/project-name/public;
    index index.php index.html index.htm index.nginx-debian.html;

    server_name YOUR.DOMAIN.COM;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
    }

    location ~ /\.ht {
            deny all;
    }
}
```

```
- sudo nginx -t
- sudo systemctl reload nginx reload Nginx
```

### Let's Encrypt (use this link https://certbot.eff.org/lets-encrypt/ubuntufocal-nginx)

```
sudo snap install core; sudo snap refresh core
sudo apt-get remove certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
sudo certbot renew --dry-run
```

### Final mod for Nginx (this file will be updated automatically, you shouldn't modify it)

```
- sudo nano /etc/nginx/sites-available/YOUR.DOMAIN.COM // just to check
```
```
server {

    root /var/www/html/project-name/public;
    index index.php index.html index.htm index.nginx-debian.html;

    server_name YOUR.DOMAIN.COM;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
    }

    location ~ /\.ht {
            deny all;
    }

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/airways-media.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/airways-media.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = YOUR.DOMAIM.COM) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    listen [::]:80;

    server_name YOUR.DOMAIM.COM;
    return 404; # managed by Certbot


}
```

```
- sudo nginx -t
- sudo ufw app list For firewall
- sudo ufw allow 'Nginx HTTPS' to add NGINX
- sudo ufw status to verify change
- sudo systemctl reload nginx reload Nginx
```

### Configure php.ini

```
- sudo nano /etc/php/8.0/fpm/php.ini
- find upload_max_filesize= in File uploads section and set it to 5M (or any other value you need)
- find post_max_size= in Data handling section and set it to 8M (or any other value you need)
```
### Redis (follow this link https://redis.io/topics/quickstart manual)

The suggested way of installing Redis is compiling it from sources as Redis has no dependencies other than a working GCC compiler and libc. Installing it using the package manager of your Linux distribution is somewhat discouraged as usually the available version is not the latest. In order to compile Redis follow these simple steps:

```
- sudo apt install make
- sudo apt install gcc // to install C compiler as well as other tools that you may need for building software from source
- sudo apt-get install tcl // to run make test (optional)
- wget http://download.redis.io/redis-stable.tar.gz
- tar xvzf redis-stable.tar.gz
- cd redis-stable
- make
- make test // optional step
- sudo make install
- redis-server // to start redis server
- redis-cli ping // response should be PONG
```

In the above example Redis was started without any explicit configuration file, so all the parameters will use the internal default. This is perfectly fine if you are starting Redis just to play a bit with it or for development, but for production environments you should use a configuration file.

```
- sudo mkdir /etc/redis
- sudo mkdir /var/redis
- sudo cp utils/redis_init_script /etc/init.d/redis_6379 // copy the init script that you'll find in the Redis distribution under the utils directory into /etc/init.d
- sudo nano /etc/init.d/redis_6379 // modify REDISPORT accordingly to the port you are using or leave 6379
- sudo cp redis.conf /etc/redis/6379.conf
- sudo mkdir /var/redis/6379
- sudo nano /etc/redis/6379.conf
```

Edit the configuration file, making sure to perform the following changes:

* Set daemonize to yes (by default it is set to no)
* Set the pidfile to /var/run/redis_6379.pid (modify the port if needed)
* Change the port accordingly
* Set your preferred loglevel
* Set the logfile to /var/log/redis_6379.log
* Set the dir to /var/redis/6379

Finally add the new Redis init script to all the default runlevels using the following command

```
- sudo update-rc.d redis_6379 defaults
- sudo /etc/init.d/redis_6379 start // to start redis server
- sudo /etc/init.d/redis_6379 stop // use this command to stop redis server
```

### Supervisor

```
- sudo apt-get install supervisor
- service supervisor status // if active use next step to kill the process
- sudo ps aux | grep supervisor // find PID here is 2957 (root        2957  0.6  2.1  31272 21928 ?        Ss   19:16   0:00 /usr/bin/python3 /usr/bin/supervisord -n -c /etc/supervisor/supervisord.conf) and kill it
- sudo kill -2957 pid
- sudo groupadd supervisor
- sudo usermod -a -G supervisor laravel // laravel is a user here
- sudo nano /etc/supervisor/supervisord.conf // in this file find and change next 2 entries below
- chmod=0770 (default 0700)
- chown=root:supervisor
- sudo service supervisor restart
- cd /etc/supervisor/conf.d // in this directory you can create any number of config files
- sudo nano mail_queue_worker.conf // create config file and add the folowwing:
```

```
[program:mail_queue_worker]
process_name=%(program_name)s_%(process_num)02d
command=sudo php /var/www/html/project-name/artisan queue:work --queue=mail_queue --tries=3 --daemon
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=root
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/html/project-name/storage/logs/mail_queue_worker.log
```

```
- sudo supervisorctl reread // response should be available
- sudo supervisorctl update
- sudo supervisorctl status // should be running
```
