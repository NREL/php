ARG FROM_TAG

FROM wodby/base-php:${FROM_TAG}

ARG PHP_DEV
ARG PHP_DEBUG

ARG PECL_HTTP_PROXY

ARG WODBY_USER_ID=1000
ARG WODBY_GROUP_ID=1000

ENV PHP_DEV="${PHP_DEV}" \
    PHP_DEBUG="${PHP_DEBUG}" \
    LD_PRELOAD="/usr/lib/preloadable_libiconv.so php" \
    PHP_FPM_USER="www-data" \
    PHP_FPM_GROUP="www-data"

ENV APP_ROOT="/var/www/html" \
    CONF_DIR="/var/www/conf" \
    FILES_DIR="/var/www/common"

ENV PATH="${PATH}:/home/wodby/.composer/vendor/bin:${APP_ROOT}/vendor/bin:${APP_ROOT}/bin" \
    SSHD_HOST_KEYS_DIR="/etc/ssh" \
    ENV="/home/wodby/.shrc" \
    \
    GIT_USER_EMAIL="drupaladmin@nrel.gov" \
    GIT_USER_NAME="Drupal Admin"

RUN set -xe; \
    \
    # Delete existing user/group if uid/gid occupied.
    existing_group=$(getent group "${WODBY_GROUP_ID}" | cut -d: -f1); \
    if [[ -n "${existing_group}" ]]; then delgroup "${existing_group}"; fi; \
    existing_user=$(getent passwd "${WODBY_USER_ID}" | cut -d: -f1); \
    if [[ -n "${existing_user}" ]]; then deluser "${existing_user}"; fi; \
    \
	addgroup -g "${WODBY_GROUP_ID}" -S wodby; \
	adduser -u "${WODBY_USER_ID}" -D -S -s /bin/bash -G wodby wodby; \
	adduser wodby www-data; \
	sed -i '/^wodby/s/!/*/' /etc/shadow; \
	\
    apk add --update --no-cache -t .wodby-php-run-deps \
        c-client=2007f-r10 \
        fcgi \
        findutils \
        freetype=2.10.0-r0 \
        git \
        gmp=6.1.2-r1 \
        icu-libs=64.2-r0 \
        imagemagick=7.0.8.58-r0 \
        jpegoptim=1.4.6-r0 \
        less \
        libbz2=1.0.6-r7 \
        libevent=2.1.10-r0 \
        libjpeg-turbo=2.0.3-r0 \
        libjpeg-turbo-utils \
        libldap=2.4.48-r0 \
        libltdl=2.4.6-r6 \
        libc6-compat \
        libmemcached-libs=1.0.18-r3 \
        libmcrypt=2.5.8-r7 \
        libpng=1.6.37-r1 \
        librdkafka=1.0.1-r1 \
        libuuid=2.33.2-r0 \
        libwebp=1.0.2-r0 \
        libxslt=1.1.33-r2 \
        libzip=1.5.2-r0 \
        make \
        mariadb-client \
        nano \
        openssh \
        openssh-client \
        patch \
        pngquant \
        postgresql-client \
        rabbitmq-c=0.8.0-r5 \
        rsync \
        su-exec \
        sudo \
        tidyhtml-libs=5.6.0-r0 \
        # todo: move out tig and tmux to -dev version.
        tig \
        tmux \
        yaml=0.2.2-r1; \
    \
    apk add --update --no-cache -t .wodby-php-build-deps \
        autoconf \
        cmake \
        build-base \
        bzip2-dev \
        freetype-dev \
        gmp-dev \
        icu-dev \
        imagemagick-dev \
        imap-dev \
        jpeg-dev \
        krb5-dev \
        libevent-dev \
        libgcrypt-dev \
        libjpeg-turbo-dev \
        libmemcached-dev \
        libmcrypt-dev \
        libpng-dev \
        librdkafka-dev \
        libtool \
        libwebp-dev \
        libxslt-dev \
        libzip-dev \
        openldap-dev \
        openssl-dev \
        pcre-dev \
        postgresql-dev \
        rabbitmq-c-dev \
        tidyhtml-dev \
        yaml-dev; \
    \
    apk add -U --no-cache -t .wodby-php-edge-run-deps -X http://dl-cdn.alpinelinux.org/alpine/edge/community/ \
        gnu-libiconv=1.15-r2; \
    \
    docker-php-source extract; \
    \
    docker-php-ext-install \
        bcmath \
        bz2 \
        calendar \
        exif \
        gmp \
        intl \
        ldap \
        mysqli \
        opcache \
        pcntl \
        pdo_mysql \
        pdo_pgsql \
        pgsql \
        soap \
        sockets \
        tidy \
        xmlrpc \
        xsl \
        zip; \
    \
    # GD
    docker-php-ext-configure gd \
        --with-gd \
        --with-webp-dir \
        --with-freetype-dir=/usr/include/ \
        --with-png-dir=/usr/include/ \
        --with-jpeg-dir=/usr/include/; \
      NPROC=$(getconf _NPROCESSORS_ONLN); \
      docker-php-ext-install "-j${NPROC}" gd; \
    \
    # IMAP
    PHP_OPENSSL=yes docker-php-ext-configure imap \
        --with-kerberos \
        --with-imap-ssl; \
    docker-php-ext-install "-j${NPROC}" imap; \
    \
    pecl config-set php_ini "${PHP_INI_DIR}/php.ini"; \
    if [[ -n "${PECL_HTTP_PROXY}" ]]; then \
        # Using pear as pecl throw errors: https://blog.flowl.info/2015/peclpear-behind-proxy-how-to/
        pear config-set http_proxy "${PECL_HTTP_PROXY}"; \
    fi; \
    # mcrypt moved to pecl in PHP 7.2
    if [[ "${PHP_VERSION:0:3}" == "7.2" ]]; then \
        pecl install mcrypt-1.0.3; \
        docker-php-ext-enable mcrypt; \
    elif [[ "${PHP_VERSION:0:3}" == "7.1" ]]; then \
        docker-php-ext-install mcrypt; \
    fi; \
    \
    if [[ "${PHP_VERSION:0:3}" != "7.4" ]]; then \
        # NewRelic extension and agent.
        newrelic_url="http://download.newrelic.com/php_agent/release/"; \
        wget -r -nd --no-parent -P /tmp/newrelic -Alinux-musl.tar.gz "${newrelic_url}" >/dev/null 2>&1; \
        tar -xzf /tmp/newrelic/newrelic-php*.tar.gz --strip=1 -C /tmp/newrelic; \
        export NR_INSTALL_SILENT=true; \
        export NR_INSTALL_USE_CP_NOT_LN=true; \
        bash /tmp/newrelic/newrelic-install install; \
        rm /usr/local/etc/php/conf.d/newrelic.ini; \
        mkdir -p /var/log/newrelic/; \
        chown -R www-data:www-data /var/log/newrelic/; \
        chmod -R 775 /var/log/newrelic/; \
        \
        \
        pecl install oauth-2.0.3; \
        docker-php-ext-enable oauth; \
    fi; \
    \
    pecl install \
        amqp-1.9.4 \
        apcu-5.1.18 \
        ast-1.0.5 \
        ds-1.2.9 \
        event-2.5.3 \
        grpc-1.23.1 \
        igbinary-3.0.1 \
        imagick-3.4.4 \
        jsmin-3.0.0 \
        memcached-3.1.4 \
        mongodb-1.6.0 \
        rdkafka-3.1.2 \
        uuid-1.0.4 \
        xdebug-2.8.0 \
        yaml-2.0.4; \
    \
    docker-php-ext-enable \
        amqp \
        apcu \
        ast \
        ds \
        event \
        igbinary \
        imagick \
        grpc \
        jsmin \
        mongodb \
        rdkafka \
        uuid \
        xdebug \
        yaml; \
    \
    # Event extension should be loaded after sockets.
    # http://osmanov-dev-notes.blogspot.com/2013/07/fixing-php-start-up-error-unable-to.html
    mv /usr/local/etc/php/conf.d/docker-php-ext-event.ini /usr/local/etc/php/conf.d/z-docker-php-ext-event.ini; \
    \
    if [[ "${PHP_VERSION:0:3}" != "7.4" ]]; then \
        # Blackfire extension (they have free tier).
        mkdir -p /tmp/blackfire; \
        version=$(php -r "echo PHP_MAJOR_VERSION.PHP_MINOR_VERSION;"); \
        blackfire_url="https://blackfire.io/api/v1/releases/probe/php/alpine/amd64/${version}"; \
        wget -qO- "${blackfire_url}" | tar xz --no-same-owner -C /tmp/blackfire; \
        mv /tmp/blackfire/blackfire-*.so $(php -r "echo ini_get('extension_dir');")/blackfire.so; \
    fi; \
    \
    # aws-elasticache-cluster-client-memcached-for-php
    mkdir -p /tmp/cluster; \
    version=$(php -r "echo PHP_MAJOR_VERSION.'.'.PHP_MINOR_VERSION;"); \
    # cluster_url="https://elasticache-downloads.s3.amazonaws.com/ClusterClient/PHP-${version}/latest-64bit"; \
    cluster_url="https://elasticache-downloads.s3.amazonaws.com/ClusterClient/PHP-7.3/latest-64bit"; \
    wget -qO- "${cluster_url}" | tar xz --no-same-owner -C /tmp/cluster; \
    mv /tmp/cluster/amazon-elasticache-cluster-client.so $(php -r "echo ini_get('extension_dir');")/; \

        docker-php-ext-enable amazon-elasticache-cluster-client; \
    # Uploadprogress.
    mkdir -p /usr/src/php/ext/uploadprogress; \
    up_url="https://github.com/wodby/pecl-php-uploadprogress/archive/latest.tar.gz"; \
    wget -qO- "${up_url}" | tar xz --strip-components=1 -C /usr/src/php/ext/uploadprogress; \
    docker-php-ext-install uploadprogress; \
    \
    # Tideways xhprof.
    xhprof_ext_ver="5.0-beta3"; \
    mkdir -p /usr/src/php/ext/tideways_xhprof; \
    xhprof_url="https://github.com/tideways/php-xhprof-extension/archive/v${xhprof_ext_ver}.tar.gz"; \
    wget -qO- "${xhprof_url}" | tar xz --strip-components=1 -C /usr/src/php/ext/tideways_xhprof; \
    docker-php-ext-install tideways_xhprof; \
    \
    wget -qO- https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer; \
    \
    # Install Walter (deprecated).
    walter_ver="1.4.0"; \
    walter_url="https://github.com/walter-cd/walter/releases/download/v${walter_ver}/walter_${walter_ver}_linux_amd64.tar.gz"; \
    wget -qO- "${walter_url}" | tar xz -C /tmp/; \
    mv /tmp/walter_linux_amd64/walter /usr/local/bin; \
    \
    { \
        echo 'export PS1="\u@${WODBY_APP_NAME:-php}.${WODBY_ENVIRONMENT_NAME:-container}:\w $ "'; \
        # Make sure PATH is the same for ssh sessions.
        echo "export PATH=${PATH}"; \
    } | tee /home/wodby/.shrc; \
    \
    cp /home/wodby/.shrc /home/wodby/.bashrc; \
    cp /home/wodby/.shrc /home/wodby/.bash_profile; \
    \
    { \
        echo 'Defaults env_keep += "APP_ROOT FILES_DIR"' ; \
        \
        if [[ -n "${PHP_DEV}" ]]; then \
            echo 'wodby ALL=(root) NOPASSWD:SETENV:ALL'; \
        else \
            echo -n 'wodby ALL=(root) NOPASSWD:SETENV: ' ; \
            echo -n '/usr/local/bin/files_chmod, ' ; \
            echo -n '/usr/local/bin/files_chown, ' ; \
            echo -n '/usr/local/bin/files_sync, ' ; \
            echo -n '/usr/local/bin/gen_ssh_keys, ' ; \
            echo -n '/usr/local/bin/init_container, ' ; \
            echo -n '/usr/local/bin/migrate, ' ; \
            echo -n '/usr/local/sbin/php-fpm, ' ; \
            echo -n '/usr/sbin/sshd, ' ; \
            echo '/usr/sbin/crond' ; \
        fi; \
    } | tee /etc/sudoers.d/wodby; \
    \
    echo "TLS_CACERTDIR /etc/ssl/certs/" >> /etc/openldap/ldap.conf; \
    \
    install -o wodby -g wodby -d \
        "${APP_ROOT}" \
        "${CONF_DIR}" \
        /home/wodby/.ssh; \
    \
    install -o www-data -g www-data -d \
        "${FILES_DIR}/files" \
        "${FILES_DIR}/files/private" \
        "${FILES_DIR}/sessions" \
        "${FILES_DIR}/xdebug/traces" \
        "${FILES_DIR}/xdebug/profiler" \
        /home/www-data/.ssh; \
    \
    chmod -R 775 "${FILES_DIR}"; \
    chown -R wodby:wodby \
        "${PHP_INI_DIR}/conf.d" \
        /usr/local/etc/php-fpm.d \
        /home/wodby/.[^.]*; \
    \
    touch /etc/ssh/sshd_config; \
    chown wodby: /etc/ssh/sshd_config; \
    \
    rm /etc/crontabs/root; \
    # deprecated: remove in favor of bind mounts.
    touch /etc/crontabs/www-data; \
    chown root:www-data /etc/crontabs/www-data; \
    chmod 660 /etc/crontabs/www-data; \
    \
    su-exec wodby composer clear-cache; \
    docker-php-source delete; \
    apk del --purge .wodby-php-build-deps; \
    pecl clear-cache; \
    \
    rm -rf \
        /usr/src/php/ext/ast \
        /usr/src/php/ext/uploadprogress \
        /usr/include/php \
        /usr/lib/php/build \
        /tmp/* \
        /root/.composer \
        /var/cache/apk/*; \
    \
    if [[ -z "${PHP_DEV}" ]]; then \
        rm -rf /usr/src/php.tar.xz; \
    fi

USER wodby

WORKDIR ${APP_ROOT}
EXPOSE 9000

COPY templates /etc/gotpl/
COPY docker-entrypoint.sh /
COPY bin /usr/local/bin/

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["sudo", "-E", "LD_PRELOAD=/usr/lib/preloadable_libiconv.so", "php-fpm"]
