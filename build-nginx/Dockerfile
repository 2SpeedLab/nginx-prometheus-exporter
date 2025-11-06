# Build stage
FROM debian:bookworm-slim AS builder

# NGINX version
ARG NGINX_VERSION=1.27.0

# Install minimal build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpcre3-dev \
    zlib1g-dev \
    libssl-dev \
    wget \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Download NGINX source
WORKDIR /tmp
RUN wget https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
    && tar -xzf nginx-${NGINX_VERSION}.tar.gz

# Configure with ONLY monitoring flags
WORKDIR /tmp/nginx-${NGINX_VERSION}
RUN ./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/run/nginx.lock \
    --user=nginx \
    --group=nginx \
    --with-http_stub_status_module \
    --with-http_ssl_module \
    --with-http_realip_module \
    --with-http_v2_module \
    --with-stream \
    --with-stream_ssl_module \
    --with-stream_realip_module \
    --with-threads \
    --with-file-aio \
    && make -j$(nproc) \
    && make install

# Runtime stage
FROM debian:bookworm-slim

LABEL maintainer="NGINX Monitoring Build"
LABEL description="NGINX with monitoring modules only"

# Install minimal runtime dependencies
RUN apt-get update && apt-get install -y \
    libpcre3 \
    zlib1g \
    libssl3 \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy NGINX binary from builder
COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /etc/nginx /etc/nginx

# Create nginx user and directories
RUN useradd -r -M -s /sbin/nologin nginx \
    && mkdir -p /var/log/nginx \
    && mkdir -p /usr/share/nginx/html \
    && chown -R nginx:nginx /var/log/nginx \
    && chown -R nginx:nginx /usr/share/nginx/html

COPY nginx.conf etc/nginx/nginx.conf
COPY index.html /usr/share/nginx/html/index.html
# Expose ports
EXPOSE 80 8080

# Forward logs to docker
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
