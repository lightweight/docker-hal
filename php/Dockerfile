FROM debian:jessie
# modified from config by MAINTAINER https://github.com/helderco/
MAINTAINER Dave Lane <dave@davelane.nz> (@lightweight) https://davelane.nz

# persistent / runtime deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-utils \
    ca-certificates \
    cron \
    curl \
    default-jre \
    git \
    imagemagick \
    less \
    librecode0 \
    libmariadb-client-lgpl-dev \
#      libmariadb-client-lgpl-dev-compat \
    libmariadb2 \
    libmysqlclient-dev \
    libmysqlclient18 \
    libpng12-0 \
    libsqlite3-0 \
    libxml2 \
    mariadb-client \
    net-tools \
    vim \
    wkhtmltopdf \
    unzip zip \
    && apt-get clean \
    && rm -r /var/lib/apt/lists/*

# phpize deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    autoconf \
    file \
    g++ \
    gcc \
    libc-dev \
    make \
    pkg-config \
    re2c \
    && apt-get clean \
    && rm -r /var/lib/apt/lists/*

ENV PHP_INI_DIR /usr/local/etc/php
RUN mkdir -p $PHP_INI_DIR/conf.d

ENV GPG_KEYS 0B96609E270F565C13292B24C13C70B87267B52D 0A95E9A026542D53835E3F3A7DEC4E69FC9C83D7 0E604491
RUN set -xe \
    && for key in $GPG_KEYS; do \
        gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
    done

# compile openssl, otherwise --with-openssl won't work
RUN OPENSSL_VERSION="1.0.2k" \
    && cd /tmp \
    && mkdir openssl \
    && curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz" -o openssl.tar.gz \
    && curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz.asc" -o openssl.tar.gz.asc \
    && gpg --verify openssl.tar.gz.asc \
    && tar -xzf openssl.tar.gz -C openssl --strip-components=1 \
    && cd /tmp/openssl \
    && ./config && make && make install \
    && rm -rf /tmp/*

ENV PHP_VERSION 5.3.29

# php 5.3 needs older autoconf
# --enable-mysqlnd is included below because it's harder to compile after the fact the extensions are (since it's a plugin for several extensions, not an extension in itself)
RUN buildDeps=" \
    autoconf2.13 \
    libcurl4-openssl-dev \
    libfreetype6-dev \
    libgd2-xpm-dev \
    libjpeg-dev \
    libpng12-0-dev \
    libreadline6-dev \
    librecode-dev \
    libsqlite3-dev \
    libssl-dev \
    libxml2-dev \
    libxpm-dev \
    xz-utils \
    zlib1g-dev \
    " \
    && set -x \
    && apt-get update && apt-get install -y $buildDeps --no-install-recommends && rm -rf /var/lib/apt/lists/* \
    && curl -SL "http://php.net/get/php-$PHP_VERSION.tar.xz/from/this/mirror" -o php.tar.xz \
    && curl -SL "http://php.net/get/php-$PHP_VERSION.tar.xz.asc/from/this/mirror" -o php.tar.xz.asc \
    && gpg --verify php.tar.xz.asc \
    && mkdir -p /usr/src/php \
    && tar -xof php.tar.xz -C /usr/src/php --strip-components=1 \
    && rm php.tar.xz* \
    && cd /usr/src/php \
    && ./configure \
    --with-config-file-path="$PHP_INI_DIR" \
    --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
    --enable-fpm \
    --with-fpm-user=www-data \
    --with-fpm-group=www-data \
    --disable-cgi \
    --enable-mbstring \
    --enable-mysqlnd \
    --with-mysql \
    --with-curl \
    --with-gd --with-jpeg-dir=/usr/lib64 \
    --with-openssl=/usr/local/ssl \
    --with-readline \
    --with-recode \
    --with-zlib \
    && make -j"$(nproc)" \
    && make install \
    && { find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; } \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false -o APT::AutoRemove::SuggestsImportant=false $buildDeps \
    && make clean

COPY docker-php-* /usr/local/bin/

#WORKDIR /var/www/html

RUN set -ex && cd /usr/local/etc && \
    if [ -d php-fpm.d ] ; then \
        # for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf" \
        sed 's!=NONE!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
        touch php-fpm.d/www.conf; \
    else \
        # PHP 5.x don't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency \
        mkdir php-fpm.d; \
        touch php-fpm.d/www.conf; \
        { \
          echo '[global]'; \
          echo 'include=etc/php-fpm.d/*.conf'; \
        } | tee php-fpm.conf; \
    fi

RUN set -ex && cd /usr/local/etc && { \
        echo '[global]'; \
        echo 'error_log = /proc/self/fd/2'; \
        echo; \
        echo '[www]'; \
        echo '; if we send this to /proc/self/fd/1, it never appears'; \
        echo 'access.log = /proc/self/fd/2'; \
        echo; \
        echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
        echo 'catch_workers_output = yes'; \
    } | tee php-fpm.d/docker.conf;

RUN set -ex && cd /usr/local/etc && { \
        echo '[www]'; \
        echo 'user = www-data'; \
        echo 'group = www-data'; \
        echo '; this does not appear to be supported by php 5.3.x'; \
        echo ';clear_env = no'; \
        #echo 'security.limit_extensions = .php'; \
        echo 'pm = ondemand'; \
        echo 'pm.max_children = 20'; \
        echo 'pm.start_servers = 2'; \
        echo 'pm.min_spare_servers = 1'; \
        echo 'pm.max_spare_servers = 3'; \
    } | tee php-fpm.d/www.conf;

RUN set -ex && cd /usr/local/etc && { \
        echo '[global]'; \
        echo 'daemonize = no'; \
        echo; \
        echo '[www]'; \
        echo 'listen = 9000'; \
    } | tee php-fpm.d/zz-docker.conf;

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

RUN apt update && apt install -y sudo

WORKDIR /var/www/html

EXPOSE 9000

# Copy init scripts and custom .htaccess
COPY docker-entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

CMD ["php-fpm"]
