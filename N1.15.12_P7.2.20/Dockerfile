FROM alpine:3.10

LABEL maintainer="Rice <rice_ad@qq.com>" NGINX_ver="1.15.12" PHP_ver="7.2.20"

ENV NGINX_VERSION 1.15.12
ENV PHP_VERSION 7.2.20
# 环境变量定义
	## NGINX
ENV CONFIG="\
		--prefix=/usr/local/nginx \
		--error-log-path=/var/log/nginx/error.log \
		--http-log-path=/var/log/nginx/access.log \
		--user=www \
		--group=www \
		--with-http_ssl_module \
		--with-http_realip_module \
		--with-http_addition_module \
		--with-http_sub_module \
		--with-http_dav_module \
		--with-http_flv_module \
		--with-http_mp4_module \
		--with-http_gunzip_module \
		--with-http_gzip_static_module \
		--with-http_random_index_module \
		--with-http_secure_link_module \
		--with-http_stub_status_module \
		--with-http_auth_request_module \
		--with-http_xslt_module=dynamic \
		--with-http_image_filter_module=dynamic \
		--with-http_geoip_module=dynamic \
		--with-threads \
		--with-stream \
		--with-stream_ssl_module \
		--with-stream_ssl_preread_module \
		--with-stream_realip_module \
		--with-stream_geoip_module=dynamic \
		--with-http_slice_module \
		--with-mail \
		--with-mail_ssl_module \
		--with-compat \
		--with-http_v2_module \
	" 
	## PHP
ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www --with-fpm-group=www --disable-cgi
ENV PHPIZE_DEPS autoconf dpkg-dev dpkg file g++ gcc libc-dev make pkgconf re2c
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"

# 拷贝预先文件
COPY src /usr/src
COPY php/docker-php-source /usr/local/bin/
COPY php/docker-php-ext-* php/nginx-php-startup /usr/local/bin/

# 编译环境准备
	## 修改alpine镜像源为阿里云和时区为Asia/Shanghai
RUN set -x \
	&& sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
	&& apk update \
	&& apk add -U tzdata \
    && cp -rf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && apk del tzdata \
    ## 安装依赖包 For Nginx
    && apk add --no-cache --virtual .build-deps \
		gcc \
		libc-dev \
		make \
		openssl-dev \
		pcre-dev \
		zlib-dev \
		linux-headers \
		curl \
		gnupg1 \
		libxslt-dev \
		gd-dev \
		geoip-dev \
# 创建用户
	&& addgroup -S www \
	&& adduser -D -S -h /var/cache/nginx -s /sbin/nologin -G www www \
# 创建目录
	&& mkdir -p /usr/src \
	&& mkdir -p /usr/local/{php,nginx/conf/conf.d} \
	&& mkdir -p /www \
	&& chown www:www /www \
	&& chmod 777 /www \
# 源码下载
	&& cd /usr/src \
	# && curl -fSL ${NGINX_URL} -o nginx.tar.gz \
	# && curl -fSL ${PHP_URL} -o php.tar.xz \
#编译nginx
	&& tar -zxC /usr/src -f nginx.tar.gz \
	&& rm nginx.tar.gz \
	&& cd /usr/src/nginx-$NGINX_VERSION \
	&& ./configure $CONFIG --with-debug \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& mv objs/nginx objs/nginx-debug \
	&& mv objs/ngx_http_xslt_filter_module.so objs/ngx_http_xslt_filter_module-debug.so \
	&& mv objs/ngx_http_image_filter_module.so objs/ngx_http_image_filter_module-debug.so \
	&& mv objs/ngx_http_geoip_module.so objs/ngx_http_geoip_module-debug.so \
	&& mv objs/ngx_stream_geoip_module.so objs/ngx_stream_geoip_module-debug.so \
	&& ./configure $CONFIG \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& make install \
	&& install -m644 html/index.html /www \
	&& install -m644 html/50x.html /www \
	&& install -m755 objs/nginx-debug /usr/sbin/nginx-debug \
	&& install -m755 objs/ngx_http_xslt_filter_module-debug.so /usr/local/nginx/modules/ngx_http_xslt_filter_module-debug.so \
	&& install -m755 objs/ngx_http_image_filter_module-debug.so /usr/local/nginx/modules/ngx_http_image_filter_module-debug.so \
	&& install -m755 objs/ngx_http_geoip_module-debug.so /usr/local/nginx/modules/ngx_http_geoip_module-debug.so \
	&& install -m755 objs/ngx_stream_geoip_module-debug.so /usr/local/nginx/modules/ngx_stream_geoip_module-debug.so \
	&& ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx \
	&& strip /usr/local/nginx/sbin/nginx \
	&& strip /usr/local/nginx/modules/*.so \
	&& rm -rf /usr/src/nginx-$NGINX_VERSION \
	&& apk add --no-cache --virtual .gettext gettext \
	&& mv /usr/bin/envsubst /tmp/ \
	&& runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' /usr/local/nginx/sbin/nginx /usr/local/nginx/modules/*.so /tmp/envsubst \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)" \
	&& apk add --no-cache --virtual .nginx-rundeps $runDeps \
	&& apk del .build-deps \
	&& apk del .gettext \
	&& mv /tmp/envsubst /usr/local/bin/ \
	&& ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log 
## 构建PHP
RUN set -x \
	&& apk add --no-cache \
		ca-certificates \
		curl \
		tar \
		xz \
		openssl \
	## 安装依赖包 For PHP
	&& apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		argon2-dev \
		coreutils \
		curl-dev \
		libedit-dev \
		libsodium-dev \
		libxml2-dev \
		openssl-dev \
		sqlite-dev \
	&& export CFLAGS="$PHP_CFLAGS"  \
			  CPPFLAGS="$PHP_CPPFLAGS" \
			  LDFLAGS="$PHP_LDFLAGS" \
	&& docker-php-source extract \
	&& cd /usr/src/php \
	&& ./configure --prefix=/usr/local/php \
			    --with-fpm-user=www \
			    --with-fpm-group=www \
			    --with-mysqli \
			    --with-pdo-mysql \
			    --with-openssl \
			    --with-gd \
			    --with-iconv \
			    --with-zlib \
			    --with-gettext \
			    --with-curl \
			    --with-png-dir \
			    --with-jpeg-dir \
			    --with-freetype-dir \
			    --with-xmlrpc \
			    --with-mhash \
			    --enable-fpm \
			    --enable-xml \
			    --enable-shmop \
			    --enable-sysvsem \
			    --enable-inline-optimization \
			    --enable-mbregex \
			    --enable-mbstring \
			    --enable-ftp \
			    --enable-mysqlnd \
			    --enable-pcntl \
			    --enable-sockets \
			    --enable-zip \
			    --enable-soap \
			    --enable-session \
			    --enable-opcache \
			    --enable-bcmath \
			    --enable-exif \
			    --enable-fileinfo \
			    --disable-rpath \
			    --enable-ipv6 \
			    --disable-debug \
			    --without-pear \
    && make \
    && make install \
	find -type f -name '*.a' -delete; \
	find /usr/local/bin /usr/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; \
	make clean; \
	cd /www \
	&& docker-php-source delete; \
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)" \
	&& apk add --no-cache $runDeps \
	&& apk del --no-network .build-deps; \
	pecl update-channels \
	&& rm -rf /tmp/pear ~/.pearrc \
	&& php --version \
	&& docker-php-ext-enable sodium

COPY nginx/nginx.conf /usr/local/nginx/conf/nginx.conf
COPY nginx/nginx.vh.default.conf /usr/local/nginx/conf/conf.d/default.conf

EXPOSE 80
STOPSIGNAL SIGTERM

WORKDIR /www

CMD ["nginx-php-startup"]