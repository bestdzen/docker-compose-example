
## Пример создания тестовой среды для разработки на WordPress

### Стек технологий:
* Nginx (proxy): 1.23.1
* Nginx (2 backends): 1.23.1
* MariaDB: 10.6.11
* WordPress (2 app): 6.1.1-php8.2-fpm
* Redis: 5.0.7-alpine
* phpMyAdmin: Latest

**Opcache** - его главная задача — единожды скомпилировать каждый PHP-скрипт 
и закэшировать получившиеся опкоды в общую память, чтобы их мог считать и
выполнить каждый рабочий процесс PHP

**Redis** — сетевое журналируемое хранилище данных типа "ключ" — "значение" с открытым исходным кодом.
При загрузке страницы необходимый SQL-запрос извлекается из памяти Redis; благодаря этому база данных не 
перегружается дублируемыми запросами. В результате страница загружается значительно быстрее, а влияние
сервера на ресурсы базы данных уменьшается. Если поступивший запрос не обнаружен в памяти Redis,
он извлекается из базы данных, после чего добавляется в кэш-память Redis.

```yaml
version: '3.3'

services:
  
  proxy:
    image: bestdzen/microcluster-proxy-prod
    hostname: proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"    
    networks:
      - microcluster_net 
    depends_on:
      - wordpress
    environment:
      - BACKEND_1=${BACKEND_1}
      - BACKEND_2=${BACKEND_2}
    command: >
      /bin/sh -c
      "envsubst '
      $${BACKEND_1} $${BACKEND_2}
      '< /etc/nginx/nginx.conf.template
      > /etc/nginx/nginx.conf
      && nginx -g 'daemon off;'"
  webserver:
    hostname: webserver    
    image: bestdzen/microcluster-backend-prod        
    restart: always 
    networks:
      - microcluster_net
    depends_on:
      - proxy
    deploy:   
      mode: replicated
      replicas: 2
    environment:
      - RESOLVER1=${RESOLVER1}
      - RESOLVER2=${RESOLVER2}
      - PUBLIC_IP=${PUBLIC_IP}
      - DOMAIN_NAME=${DOMAIN_NAME}
    command: >
      /bin/sh -c
      "envsubst '
      $${RESOLVER1} $${RESOLVER2} $${PUBLIC_IP} $${DOMAIN_NAME}
      '< /etc/nginx/sites-enabled/backend-https.template
      >  /etc/nginx/sites-enabled/backend-https.conf
      && nginx -g 'daemon off;'"
    volumes:
      # dev
      #- ./dev_html:/var/www/html
      # prod
      - app_pub:/var/www/html
  wordpress:
    image: bestdzen/microcluster-wordpress    
    hostname: wordpress    
    depends_on:
      - db
    restart: always
    networks:
      - microcluster_net    
    environment:
      - WORDPRESS_DB_HOST=${WORDPRESS_DB_HOST}
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - WORDPRESS_DB_NAME=${WORDPRESS_DB_NAME}
      - WORDPRESS_DB_USER=${WORDPRESS_DB_USER}
      - WORDPRESS_DB_PASSWORD=${WORDPRESS_DB_PASSWORD}
      - WORDPRESS_TABLE_PREFIX=${WORDPRESS_TABLE_PREFIX}
    deploy:      
      replicas: 2      
    volumes:
      # dev
      #- ./dev_html:/var/www/html
      # prod
      - app_pub:/var/www/html 
  db:
    image: bestdzen/microcluster-db-prod    
    hostname: db    
    restart: always    
    networks:
      - microcluster_net    
    volumes:   
      - db-data_pub:/var/lib/mysql 
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}      
      MARIADB_CHARACTER_SET_SERVER: ${MARIADB_CHARACTER_SET_SERVER}
      MARIADB_COLLATION_SERVER: ${MARIADB_COLLATION_SERVER}
      MARIADB_DEFAULT_CHARACTER_SET: ${MARIADB_DEFAULT_CHARACTER_SET}
      MARIADB_INNODB_BUFFER_POOL_SIZE: ${MARIADB_INNODB_BUFFER_POOL_SIZE}
      MARIADB_KEY_BUFFER_SIZE: ${MARIADB_KEY_BUFFER_SIZE}
      MARIADB_MYISAM_SORT_BUFFER_SIZE: ${MARIADB_MYISAM_SORT_BUFFER_SIZE}      
      TZ: ${TZ}
  redis:
    image: redis:5.0.7-alpine
    depends_on:
      - webserver
    hostname: redis    
    restart: always    
    command: redis-server --bind redis --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru --appendonly yes
    networks:
      - microcluster_net
  pma:
    image: phpmyadmin/phpmyadmin    
    restart: always
    networks:
      - microcluster_net
    ports:
      - 7777:80
    environment:
      - PMA_HOST=db
      - MYSQL_USERNAME=${MYSQL_USERNAME}
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
volumes: 
  app_pub:
  db-data_pub:  
  #dbredis_data:   
networks:
  microcluster_net:
    driver: bridge
    ipam:
      config:
        - subnet: ${LOCAL_NETWORK}
