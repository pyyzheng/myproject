# Nginx Configuration Analysis for Multiple SPA Applications

## Current Configuration Issues

### 1. **Location Block Priority Problem**
The main issue is with the location block order and specificity:

```nginx
location / {
    root /home/zpy2025/mnt/sbxbooth/app/webapp/sbxboothweb;
    index index.html 200.html;
    try_files $uri $uri/ @router;
}

location /ca/ {
    root /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
    index index.html 200.html;
    try_files $uri $uri/ 200.html;
}
```

**Problem**: The `/ca/` location block has incorrect root configuration. When accessing `/ca/`, Nginx will look for files at:
`/home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/ca/` (root + URI path)

This is wrong because it adds `/ca/` to the root path.

### 2. **SPA Routing Inconsistency**
- Main app uses `@router` fallback that redirects to `/200.html`
- `/ca/` app directly falls back to `200.html`
- This creates inconsistent behavior for SPA routing

### 3. **Static Resource Caching Conflicts**
Static resource location blocks will apply globally, which might cause issues if both apps have different caching requirements.

## Recommended Fixed Configuration

```nginx
# ======================
# 1. 主站服务配置 (www.soundbox-pod.com)
# ======================

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name www.soundbox-pod.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS 配置
server {
    listen 443 ssl http2;
    server_name www.soundbox-pod.com;

    # SSL 证书配置
    ssl_certificate /home/zpy2025/mnt/sbxbooth/app/webapp/cert/soundboxpod.com.pem;
    ssl_certificate_key /home/zpy2025/mnt/sbxbooth/app/webapp/cert/soundboxpod.com.key;

    # 安全增强配置
    ssl_session_timeout 24h;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets on;
    ssl_early_data on;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
    ssl_ecdh_curve X25519:prime256v1:secp384r1;
    ssl_dhparam /home/zpy2025/mnt/sbxbooth/app/webapp/cert/dhparam.pem;
    ssl_buffer_size 4k;
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";

    # API 代理 (放在最前面，优先级最高)
    location /soundbox/ {
        proxy_pass http://47.237.98.37:8097/soundbox/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://47.237.98.37:8097/soundbox/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # CA 应用的静态资源缓存 (更具体的路径)
    location ~* ^/ca/.*\.(css|js)$ {
        alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }

    location ~* ^/ca/.*\.(jpg|jpeg|png|gif|ico)$ {
        alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }

    location ~* ^/ca/.*\.(svg|woff|woff2|ttf|eot)$ {
        alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
        expires 365d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }

    # CA 应用配置 (使用 alias 而不是 root)
    location /ca/ {
        alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
        index index.html;
        try_files $uri $uri/ @ca_router;
    }

    # CA 应用的 SPA 路由回退
    location @ca_router {
        alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
        try_files /index.html =404;
    }

    # 主应用的静态资源缓存
    location ~* \.(html|htm)$ {
        expires 1h;
        add_header Cache-Control "public, must-revalidate";
        etag on;
        add_header Last-Modified $date_gmt;
    }

    location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }

    location ~* \.(jpg|jpeg|png|gif|ico)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }

    location ~* \.(svg|woff|woff2|ttf|eot)$ {
        expires 365d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }

    # 主应用配置 (放在最后，作为默认处理)
    location / {
        root /home/zpy2025/mnt/sbxbooth/app/webapp/sbxboothweb;
        index index.html;
        try_files $uri $uri/ @main_router;
    }

    # 主应用的 SPA 路由回退
    location @main_router {
        root /home/zpy2025/mnt/sbxbooth/app/webapp/sbxboothweb;
        try_files /index.html =404;
    }

    # 安全头优化
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
}
```

## Key Changes Made

### 1. **Fixed Location Block Order**
- API proxies moved to the top (highest priority)
- CA app static resources with specific paths
- CA app location block
- Main app static resources
- Main app location block (catches everything else)

### 2. **Used `alias` Instead of `root` for CA App**
```nginx
# Wrong (your original)
location /ca/ {
    root /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
    # This looks for files at: /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/ca/
}

# Correct (fixed)
location /ca/ {
    alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
    # This looks for files at: /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/
}
```

### 3. **Consistent SPA Routing**
- Both apps now use proper named location blocks for SPA routing
- Both fall back to their respective `index.html` files

### 4. **Separated Static Resource Handling**
- CA app static resources use specific regex patterns
- Main app static resources handle everything else

### 5. **Added Missing Headers**
- Added `X-Forwarded-Proto` to proxy configurations

## Testing the Configuration

After applying this configuration:

1. **Test CA app**: `https://www.soundbox-pod.com/ca/` should serve from `/home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/`
2. **Test main app**: `https://www.soundbox-pod.com/` should serve from `/home/zpy2025/mnt/sbxbooth/app/webapp/sbxboothweb`
3. **Test SPA routing**: Both apps should properly handle client-side routing
4. **Test static resources**: CSS, JS, images should load correctly for both apps

## Additional Recommendations

1. **Consider using different subdomains** if the applications are completely separate (e.g., `ca.soundbox-pod.com`)
2. **Add specific error pages** for each application
3. **Consider rate limiting** for API endpoints
4. **Add request logging** for debugging if needed