# Nginx Configuration Changes Summary

## 主要修改内容

### 1. 路径映射配置

已正确配置以下路径映射：

- **www.soundbox-pod.com** → `/home/zpy2025/mnt/sbxbooth/app/webapp/sbxboothweb`
- **www.soundbox-pod.com/ca** → `/home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb`

### 2. 关键修正

#### A. /ca 路径处理优化
```nginx
# 使用 ^~ 修饰符确保 /ca 路径优先匹配
location ^~ /ca/ {
    alias /home/zpy2025/opt/sbxboot/ca/webapp/sbxboothweb/;
    try_files $uri $uri/ @ca_fallback;
    index index.html 200.html;
    
    # 为 /ca 路径下的静态资源应用相同的缓存策略
    location ~* ^/ca/.*\.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
        etag on;
    }
    
    location ~* ^/ca/.*\.(jpg|jpeg|png|gif|ico)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        add_header Vary Accept;
        access_log off;
        etag on;
    }
}

# /ca 路径的回退处理
location @ca_fallback {
    rewrite ^/ca/.*$ /ca/200.html last;
}
```

#### B. GeoIP 重定向逻辑优化
```nginx
location / {
    # 国家重定向逻辑 - 只对根路径进行重定向
    if ($uri = "/") {
        # 中国用户直接显示默认页面，不重定向
        if ($geoip2_data_country_code = "CN") {
            break;
        }
        # 其他国家根据映射进行重定向
        if ($country_path != "") {
            return 302 $country_path;
        }
    }
    
    # 其他处理逻辑...
}
```

### 3. 配置要点

1. **优先级匹配**: 使用 `^~` 修饰符确保 `/ca/` 路径优先于根路径 `/` 进行匹配
2. **别名配置**: 使用 `alias` 指令将 `/ca/` 请求映射到加拿大版本的文件目录
3. **SPA 支持**: 为 `/ca/` 路径配置了独立的回退处理 `@ca_fallback`
4. **静态资源缓存**: 为 `/ca/` 路径下的静态资源配置了相同的缓存策略
5. **GeoIP 重定向**: 只在访问根路径时进行国家重定向，避免影响已经在 `/ca/` 路径的用户

### 4. 部署建议

1. **备份现有配置**: 在应用新配置前备份当前的 nginx 配置文件
2. **语法检查**: 使用 `nginx -t` 检查配置语法
3. **分阶段部署**: 建议先在测试环境验证配置
4. **监控日志**: 部署后监控访问日志确认路径映射正确工作

### 5. 测试验证

部署后可以通过以下方式验证配置：

1. 访问 `www.soundbox-pod.com` 确认显示默认版本
2. 访问 `www.soundbox-pod.com/ca` 确认显示加拿大版本
3. 访问 `www.soundbox-pod.com/geoip-test` 查看 GeoIP 信息
4. 检查不同国家 IP 的重定向行为

## 配置文件位置

完整的配置文件已保存为：`/workspace/soundbox-pod.conf`