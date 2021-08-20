# wb7-aplicativos-magento
Instalamos lo necesario:
```
apt update
apt install php7.3 php7.3-fpm php7.3-common php7.3-mbstring php7.3-xmlrpc php7.3-soap php7.3-gd php7.3-xml php7.3-intl php7.3-mysql php7.3-cli php7.3-ldap php7.3-zip php7.3-curl php7.3-bcmath php7.3-imagick php7.3-xsl php7.3-intl -y
apt install nginx python-certbot-nginx mariadb-server -y

mysql_secure_installation
mysql
	create database magento;
	grant all privileges on magento.* to 'magento_user'@'localhost' identified by 'vahKei4ovah0ieLasohaceiTahxeelae';
	flush privileges;

wget https://getcomposer.org/download/1.10.17/composer.phar -O /usr/bin/composer
chmod +x /usr/bin/composer

apt install git
cd /var/www
git clone https://github.com/magento/magento2.git
cd magento2/
git checkout 2.3.5
composer install
find var generated vendor pub/static pub/media app/etc -type f -exec chmod g+w {} +
find var generated vendor pub/static pub/media app/etc -type d -exec chmod g+ws {} +
chown -R www-data: /var/www/magento2
chmod u+x bin/magento
```
Ya podemos acceder a nuestra tienda y completar la instalación:
https://shop.devops-alumno08.com/admin_1xppeq/ 
Usuario admin: enrique

> Pregunta 1: por defecto magento2 es muy lento. Instala redis server y configura magento2 para cachear:
```
apt install -y redis
bin/magento setup:config:set --page-cache=redis --page-cache-redis-server=127.0.0.1 --page-cache-redis-db=1 --cache-backend-redis-db=0 --cache-backend=redis
```

Verificamos el fichero *app/etc/env.php* en busca de cache-> frontend y cache->page_cache que esté configurado con redis
Verificamos con el monitor de redis. Si al cargar la página salen datos:
	
`redis-cli monitor`

> Pregunta 2 : configura opcache para magento2.
Modificamos el fichero */etc/php/7.3/fpm/conf.d/10-opcache.ini*
```
[opcache]
opcache.memory_consumption=512MB
opcache.max_accelerated_files=60000
opcache.consistency_checks=0
opcache.validate_timestamps=0
opcache.enable_cli=1
```
> Pregunta 3 : configura adecuadamente un certificado ssl y la configuración de magento para utilizarlo.

Con `certbot --nginx`
> Pregunta 4 : busca una forma de consultar el estado de la cache por consola y de flushearlo.

Con `redis-cli monitor` y para borrar `redis-cli flushdb` y `redis-cli flushall`

> Pregunta 5 : Desplega un centos 8 con bookstack, nginx, php-fpm y mariadb. Securízalo adecuadamente. Documenta el proceso. la instalación de bookstack debe ser manual (sin docker).
```
yum update -y && yum install -y mariadb nginx
dnf module install php:remi-7.3
REMIRPM="http://rpms.remirepo.net/enterprise/8/remi/x86_64/remi-release-8.1-2.el8.remi.noarch.rpm"
yum install -y $REMIRPM
 
curl -LO https://raw.githubusercontent.com/blogmotion/bm-bookstack-install/master/bookstack-install-centos8.sh
sed -i 's/php72/php73/g' bookstack-install-centos8.sh
chmod +x bookstack-install-centos8.sh && ./bookstack-install-centos8.sh
* 1 * PLEASE NOTE the MariaDB password root:KMzpnKJOkNOBcE
* 2 * Logon http://centos-bookstack or http://::1 127.0.0.1 -> admin@admin.com:password
```
> Pregunta 6 : Nos han pedido desplegar un phpmyadmin para acceder a la base de datos. Instálalo y luego securiza adecuadamente el acceso en el webserver, permitiendo solo conexiones desde una IP o configurando un htpasswd delante.
```
curl -LO https://files.phpmyadmin.net/phpMyAdmin/4.9.2/phpMyAdmin-4.9.2-all-languages.zip
unzip phpMyAdmin-4.9.2-all-languages.zip
mv phpMyAdmin-4.9.2-all-languages /usr/share/phpmyadmin
cd /usr/share/phpmyadmin
mv config.sample.inc.php config.inc.php
vi config.inc.php #(cambiamos el password)
mysql < /usr/share/phpmyadmin/sql/create_tables.sql -u root -p
mkdir /usr/share/phpmyadmin/tmp
chown -R apache:apache /usr/share/phpmyadmin
chmod 777 /usr/share/phpmyadmin/tmp
ln -s $(pwd) /var/www/phpmyadmin
```
Creamos un el fichero */etc/nginx/conf.d/phpmyadmin.conf*:
```
server {
  listen 80;

  #HTTP conf:
  #listen 443 ssl;
  #ssl_certificate /etc/pki/tls/blogmotion/monserveur.crt;
  #ssl_certificate_key /etc/pki/tls/blogmotion/monserveur.key;
  #ssl_protocols TLSv1.2;
  #ssl_prefer_server_ciphers on;

  server_name db.devops-alumno08.com;

  root /var/www/phpmyadmin;

  access_log  /var/log/nginx/phpmyadmin_access.log;
  error_log  /var/log/nginx/phpmyadmin_error.log;

  client_max_body_size 1G;
  fastcgi_buffers 64 4K;

  index  index.php;

  location / {
    try_files $uri $uri/ /index.php?$query_string;
  }

  location ~ ^/(?:\.htaccess|data|config|db_structure\.xml|README) {
    deny all;
  }

  location ~ \.php(?:$|/) {
    allow 95.127.182.17;
    deny all;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_pass unix:/var/run/php-fpm.sock;
  }

  location ~* \.(?:jpg|jpeg|gif|bmp|ico|png|css|js|swf)$ {
    expires 30d;
    access_log off;
  }
}
```

Para permitir conexiones desde una IP específica tenemos que añadir a la config de nginx la orden `allow <nuestra_ip>; deny all;`.

> Pregunta 7 : Nuestro programador prefiere un editor más rústico tipo markdown. Busca una forma de cambiar el editor por defecto y que use markdown como sintáxis.

1. Within your BookStack instance, find and click on Settings in the navbar.
2. Scroll down to the Customization section.
3. Find the Page Editor setting and select Markdown from the dropdown menu.
4. Save settings.

> Pregunta 8 : Prueba de crear alguna shelve, book y page y subir alguna imagen de por lo menos 6mb. Te puedes ayudar con alguna cheatsheet de markdown.

Hemos creado una página en un libro dentro de un estante. Mediante markdown hemos copiado una imagen y pegado directamente en el editor. Dependiendo de los parámetros que tengamos configurados en php es posible que tengamos un error al pegar imágenes grandes y por lo tanto tendremos que configurar los parámetros adecuadamente:
![image](https://user-images.githubusercontent.com/65896169/130271191-197c881b-ef66-45bf-902c-65ef6fababb9.png)

Por ejemplo estos valores en */etc/php.ini*:
```
post_max_size = 10M
upload_max_filesize = 10M
```

> Pregunta 9 : Permísos. Crea una shelve a parte solo para usuarios. A continuación crea alguna página. Crea un usuario que solo pueda visualizar el shelve creado en esta pregunta y que no pueda editar ningún libro.

