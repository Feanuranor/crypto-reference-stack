# Copyright (C) 2021 Intel Corporation
# SPDX-License-Identifier: Apache-2.0

# lo anterior que tenia era centos:centos8.3.2011

FROM quay.io/centos/centos:stream8 AS compile-stage

RUN dnf -y install epel-release && \
    dnf -y update && \
    dnf group -y install "Development Tools" && \
    dnf -y install \
      cpuid \
      cmake \
      openssl-devel \
      wget \
      pcre \
      pcre-devel

#el -y asume yes en todo
RUN dnf -y --enablerepo=powertools install nasm
RUN dnf -y install perl-IPC-Cmd

ARG OPENSSL_RELEASE="OpenSSL_1_1_1k"
RUN git clone -b $OPENSSL_RELEASE https://github.com/openssl/openssl.git
RUN cd /openssl && \
    ./config enable-ec_nistp_64_gcc_128 && \
    make && \
    make install_sw

ARG IPP_CRYPTO_VERSION="ippcp_2021.3"
RUN git clone -b $IPP_CRYPTO_VERSION https://github.com/intel/ipp-crypto.git && \
    cd /ipp-crypto/sources/ippcp/crypto_mb && \
    cmake . -B"../build" \
      -DOPENSSL_INCLUDE_DIR=/usr/local/include/openssl \
      -DOPENSSL_LIBRARIES=/usr/local/lib64 \
      -DOPENSSL_ROOT_DIR=/usr/local/bin/openssl && \
    cd ../build && \
    make crypto_mb && \
    make install

ARG IPSEC_MB_VERSION="v1.0"
RUN git clone -b $IPSEC_MB_VERSION https://github.com/intel/intel-ipsec-mb && \
    cd /intel-ipsec-mb && \
    make -j 2 SAFE_DATA=y SAFE_PARAM=y SAFE_LOOKUP=y && \
    make install NOLDCONFIG=y PREFIX=/usr/local/

ARG QAT_ENGINE_VERSION="v0.6.7"
RUN git clone -b $QAT_ENGINE_VERSION https://github.com/intel/QAT_Engine.git && \
    cd /QAT_Engine && \
    ./autogen.sh && \
    ./configure \
      --with-openssl_install_dir=/usr/local/ \
      --with-qat_sw_install_dir=/usr/local/ \
      --enable-qat_sw && \
    make && \
    make install

ARG ASYNC_NGINX_VERSION="v0.4.5"
RUN git clone -b $ASYNC_NGINX_VERSION https://github.com/intel/asynch_mode_nginx.git && \
    cd /asynch_mode_nginx && \
    ./configure \
      --prefix=/var/www \
      --conf-path=/usr/local/share/nginx/conf/nginx.conf \
      --sbin-path=/usr/local/bin/nginx \
      --pid-path=/run/nginx.pid \
      --lock-path=/run/lock/nginx.lock \
      --modules-path=/var/www/modules/ \
      --without-http_rewrite_module \
      --with-http_ssl_module \
      --with-pcre \
      --add-dynamic-module=modules/nginx_qat_module/ \
      --with-cc-opt="-DNGX_SECURE_MEM -I/usr/local/include/openssl -Wno-error=deprecated-declarations -Wimplicit-fallthrough=0" \
      --with-ld-opt="-Wl,-rpath=/usr/local/lib64 -L/usr/local/lib64" \
      --user=nginx \
      --group=nginx && \
    make && \
    make install

#lo de antes centos:centos8.3.2011
FROM quay.io/centos/centos:stream8 AS runnable

COPY --from=compile-stage /usr/local/ /usr/local/
COPY --from=compile-stage /var/www/ /var/www/

COPY --chown=1000:1000 main.sh /
RUN /bin/bash -c 'chmod +x /main.sh'

COPY nginx-script.sh /nginx-script.sh
COPY nginx-avg.conf /usr/local/share/nginx/conf/nginx-avg.conf

ENV OPENSSL_ENGINES=/usr/local/lib/engines-1.1

ARG SSL_ASYNC
ENV SSL_ASYNC="${SSL_ASYNC}"

ARG NGINX_WP
ENV NGINX_WP="${NGINX_WP}"

ARG NGINX_CIPHER
ENV NGINX_CIPHER="${NGINX_CIPHER}"

ARG NGINX_HTTP
ENV NGINX_HTTP="${NGINX_HTTP}"

STOPSIGNAL SIGTERM

RUN useradd -ms /bin/bash nginx && \
    chown -R nginx:nginx /usr/local/share/nginx/ && \
    chown -R nginx:nginx /var/www/
USER nginx

ENTRYPOINT ["/main.sh"]
CMD ["/main.sh", "${SSL_ASYNC}", "${NGINX_WP}", "${NGINX_CIPHER}", "${NGINX_HTTP}"]
