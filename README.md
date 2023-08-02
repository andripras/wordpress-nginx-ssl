# wordpress-nginx-ssl

First you must create environtment (.env) file for mysql databases

```
MYSQL_ROOT_PASSWORD=root_password
MYSQL_USER=user
MYSQL_PASSWORD=password
```
Setup nginx.conf file
```
server {
        listen 80;
        listen [::]:80;

        server_name your_domain www.your_domain;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                rewrite ^ https://$host$request_uri? permanent;
        }
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name your_domain www.your_domain;

        index index.php index.html index.htm;
        
        root /var/www/html;
        
        server_tokens off;

        ssl_certificate /your/path/certs/ssl.crt;
        ssl_certificate_key /your/path/privatekey/ssl.key;

        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
        # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        # enable strict transport security only if you understand the implications

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }
        
        location = /favicon.ico { 
                log_not_found off; access_log off; 
        }
        location = /robots.txt { 
                log_not_found off; access_log off; allow all; 
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
		client_max_body_size 100M;
}
```
Create file upload.ini for upload content on wordpress.
```
file_uploads = On
memory_limit = 64M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600;
max_input_vars = 2000;
```
and then, create docker-compose.yml file.
```
version: '3' 
services:
  database:
    image: mysql:8.0
    container_name: database
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes:
      - ./database:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network
  
  wordpress:
    depends_on:
      - database
    image: wordpress:5.8-php7.4-fpm
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=database:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - ./wordpress:/var/www/html
    networks:
      - app-network
  
  phpmyadmin:
    depends_on:
      - database
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    restart: always
    ports:
      #- '8080:80'
      - '8081:80'
    environment:
      PMA_HOST: database
      MYSQL_ROOT_PASSWORD: 1nfr4t34m
    #env_file: .env
    #environment:
      #- MYSQL_DATABASE=wordpress
    volumes:
      - ./conf.php/upload.ini:/usr/local/etc/php/conf.d/upload.ini
    networks:
      - app-network
  
  nginx:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - ./ssl:/etc/ssl
    networks:
      - app-network 

networks:
  app-network:
    driver: bridge
```
compile docker-compose file using **docker-compose up -d**
