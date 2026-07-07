FROM php:8.3-apache

RUN docker-php-ext-install mysqli pdo_mysql \
  && a2enmod rewrite ssl \
  && echo "ServerName localhost" > /etc/apache2/conf-available/servername.conf \
  && a2enconf servername
