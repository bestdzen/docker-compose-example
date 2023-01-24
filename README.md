Пример создания тестовой среды для разработки на WordPress
reverse proxy с балансировкой нагрузки, ssl path-throw в контейнерах Docker.

Запуск:
docker-compose --compatibility --env-file .env -f docker-compose.pub.yml up -d
Останов:
docker-compose --compatibility --env-file .env -f docker-compose.pub.yml down 


 
                                         |----------------|
                                         | Nginx balancer | http 1.1
                                         |----------------|
                                                 / \ Балансировка нагрузки на web серверы (round-robin)
                                  |--------------------------------|            
                                  | Nginx backend-1 Nginx backend-2| http 2
                                  |--------------------------------|
     |----------|                          /   \ Балансировка нагрузки для FastCGI (round-robin ). Fastcgi кеш для WordPress
     |phpMyAdmin|      |-------------------------------------------------------|
     |----------|      |------- wordpress (php-fpm) wordpress (php-fpm)--------|
                       |---------------php opcache   php opcache --------------|
                                  /            \
                         |-------|        |---------|               
                         | Redis |        | MariaDB |
                         |-------|        |---------|
                         
                                                   
