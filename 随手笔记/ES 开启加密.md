# ES 开启加密

## 1. 加密方案

### 1.1 elasticsearch.yml

```properties
# 开启加密
xpack.security.enabled: true
# 开启传输ssl限制
xpack.security.transport.ssl.enabled: true
```

### 1.2 docker-compose.yml 配置

```yaml
services:
  es_single:
    environment: 
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
```



## 2. 命令配置密码

重启es后配置生效，es包bin目录下执行以下命令初始化密码

```shell
./elasticsearch-setup-passwords interactive
```

