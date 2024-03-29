FROM composer as composer

COPY database/ /app/database/
COPY composer.json /app/

RUN set -x ; cd /app \
      && composer install \
           --ignore-platform-reqs \
           --no-interaction \
           --no-plugins \
           --no-scripts \
           --prefer-dist

FROM php:8.0-fpm-alpine as final

MAINTAINER waynelone@qq.com

RUN apk add supervisor autoconf g++ make openssl-dev tzdata
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" \
    && docker-php-ext-install mysqli pdo_mysql sockets fileinfo bcmath pcntl \
    && pecl install redis && docker-php-ext-enable redis

ENV TZ=Asia/Shanghai
RUN ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

COPY ./supervisor.conf /etc/supervisor/supervisord.conf
COPY ./crontab.txt /var/crontab/v2board.txt
RUN /usr/bin/crontab /var/crontab/v2board.txt

VOLUME ["/var/www/html/v2board"]

WORKDIR /var/www/html/v2board

COPY . /var/www/html/v2board
COPY ./.env.example /var/www/html/v2board/
COPY --from=composer /app/vendor/ /var/www/html/v2board/vendor/
RUN chown -R www-data:www-data /var/www/html/v2board && chmod -R 777 /var/www/html/v2board

USER www-data

EXPOSE 9000

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]