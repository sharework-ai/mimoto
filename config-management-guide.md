# Mimoto 配置管理指南

## 配置加载层次结构

Mimoto 使用 Spring Boot 的多层级配置加载机制：

```
1. mosip-config/mimoto-default.properties     # 最底层：MOSIP 平台默认配置
2. inji-config/mimoto-default.properties      # 中间层：Inji 生态默认配置
3. docker-compose/config/mimoto-default.properties  # 环境层：本地部署配置
4. 环境变量                                   # 最顶层：运行时覆盖
```

## 配置文件解析

### Bootstrap 配置 (`mimoto-bootstrap.properties`)

```properties
# 配置服务器位置 - Nginx 作为配置文件服务器
spring.cloud.config.uri=http://nginx/

# 要加载的配置文件列表 - Spring 会自动查找这些文件
spring.cloud.config.name=mimoto,inji

# 应用名称
spring.application.name=mimoto

# 配置服务器文件存储路径
config.server.file.storage.uri=http://nginx/
```

### Spring Boot 配置加载顺序

1. **bootstrap.properties** 最先加载，定义配置服务器位置
2. 根据 `spring.cloud.config.name` 加载对应的配置文件
3. 按照配置文件优先级进行合并和覆盖

## 配置重载方案

### 方案 1: 创建环境覆盖文件 (推荐)

#### 优势
- 保持默认配置完整性
- 只覆盖必要的配置项
- 便于环境切换
- 降低配置遗漏风险

#### 实施步骤

1. **创建环境覆盖配置文件**

```bash
# 开发环境
cat > docker-compose/config/mimoto-dev.properties << 'EOF'
# 环境特定的配置覆盖
mosip.api.public.url=http://localhost:8099
mosip.inji.web.url=http://localhost:3004
mosip.data.share.url=http://localhost:8097

# 数据库配置
spring.datasource.url=jdbc:postgresql://localhost:5432/inji_mimoto
spring.datasource.username=postgres
spring.datasource.password=postgres

# 日志级别
logging.level.io.mosip=DEBUG
logging.level.io.mosip.mimoto=DEBUG
EOF

# 生产环境
cat > docker-compose/config/mimoto-prod.properties << 'EOF'
# 生产环境特定配置
mosip.api.public.url=https://mimoto.example.com
mosip.inji.web.url=https://injiweb.example.com
mosip.data.share.url=https://datashare.example.com

# 生产数据库
spring.datasource.url=jdbc:postgresql://prod-db:5432/inji_mimoto
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}

# 生产日志级别
logging.level.io.mosip=INFO
logging.level.io.mosip.mimoto=INFO
EOF
```

2. **修改 docker-compose.yml**

```yaml
mimoto-service:
  environment:
    # 激活多个 profile
    - active_profile_env=default,dev
    - SPRING_CONFIG_ADDITIONAL_LOCATION=/home/mosip/config/
  volumes:
    # 基础配置
    - ./config/mimoto-default.properties:/home/mosip/mimoto-default.properties
    # 环境覆盖配置
    - ./config/mimoto-dev.properties:/home/mosip/config/mimoto-dev.properties
    # Bootstrap 配置
    - ./config/mimoto-bootstrap.properties:/home/mosip/mimoto-bootstrap.properties
```

### 方案 2: 使用环境变量

#### 优势
- 支持动态配置
- 适合敏感信息
- Kubernetes 友好

#### 实施示例

```yaml
mimoto-service:
  environment:
    # 基础配置
    - active_profile_env=default
    # 环境变量覆盖
    - mosip_api_public_url=http://localhost:8099
    - mosip_inji_web_url=http://localhost:3004
    - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/inji_mimoto
    - SPRING_DATASOURCE_USERNAME=postgres
    - SPRING_DATASOURCE_PASSWORD=postgres
```

### 方案 3: 运行时配置刷新

#### 优势
- 无需重启服务
- 支持动态更新配置

#### 实施步骤

1. **在配置类上添加 @RefreshScope**

```java
@Configuration
@RefreshScope  // 支持配置动态刷新
public class Config {
    @Value("${mosip.inji.web.url}")
    private String injiWebUrl;

    // 配置会自动刷新
}
```

2. **启用 Actuator 端点**

在 `mimoto-bootstrap.properties` 中确认：
```properties
management.endpoints.web.exposure.include=info,health,refresh
```

3. **触发配置刷新**

```bash
# 修改配置文件后，发送 POST 请求刷新配置
curl -X POST http://localhost:8099/v1/mimoto/actuator/refresh
```

## 完整配置检查清单

### 从 mosip-config 继承的配置

- [ ] MOSIP 平台服务 URLs (Kernel, AuthManager, etc.)
- [ ] Keycloak 配置
- [ ] WebSub 配置
- [ ] Token 生成配置

### 从 inji-config 继承的配置

- [ ] Inji 生态特定配置
- [ ] 认证类型配置
- [ ] 缓存配置
- [ ] VC 下载配置

### 环境特定配置

- [ ] 数据库连接
- [ ] 服务端点 URLs
- [ ] OAuth2 配置
- [ ] 日志级别
- [ ] CORS 配置

### 配置文件模板

#### 完整配置检查脚本

```bash
#!/bin/bash

echo "检查 Mimoto 配置完整性..."

# 检查必需的配置项
required_configs=(
    "mosip.api.public.url"
    "mosip.inji.web.url"
    "spring.datasource.url"
    "mosip.openid.issuers"
    "mosip.openid.verifiers"
)

# 检查配置文件存在性
config_files=(
    "docker-compose/config/mimoto-default.properties"
    "docker-compose/config/mimoto-bootstrap.properties"
    "docker-compose/config/mimoto-issuers-config.json"
    "docker-compose/config/mimoto-trusted-verifiers.json"
)

for file in "${config_files[@]}"; do
    if [ -f "$file" ]; then
        echo "✓ $file 存在"
    else
        echo "✗ $file 缺失"
    fi
done

# 检查配置项
for config in "${required_configs[@]}"; do
    if grep -q "$config" docker-compose/config/mimoto-default.properties; then
        echo "✓ $config 已配置"
    else
        echo "✗ $config 未配置"
    fi
done
```

## 最佳实践

1. **保持默认配置完整**
   - 不要修改 mosip-config 和 inji-config
   - 使用覆盖文件进行环境定制

2. **配置分层管理**
   - 基础配置 → 环境配置 → 运行时配置

3. **敏感信息管理**
   - 使用环境变量或密钥管理系统
   - 不要在配置文件中硬编码密码

4. **配置验证**
   - 启动前验证所有必需配置
   - 使用配置检查脚本

5. **版本控制**
   - 配置文件纳入版本控制
   - 使用配置文件版本标签

## 故障排查

### 配置未生效

1. 检查配置文件加载顺序
2. 确认配置项名称正确
3. 查看启动日志中的配置信息

### 配置冲突

1. 检查多个配置文件中的同一配置项
2. 确认环境变量优先级
3. 使用 `@Value` 注解调试配置值

### Nginx 配置访问

1. 确认 Nginx 容器正常运行
2. 检查卷挂载路径
3. 验证配置文件权限

## 相关文件

- `mimoto/docker-compose/nginx.conf` - Nginx 配置服务器配置
- `mimoto/docker-compose/docker-compose.yml` - 服务编排配置
- `mimoto/src/main/resources/bootstrap.properties` - Spring Bootstrap 配置
- `mosip-config/mimoto-default.properties` - MOSIP 平台默认配置
- `inji-config/mimoto-default.properties` - Inji 生态默认配置

## Kubernetes 环境配置

### 当前环境架构

项目已部署在多命名空间 Kubernetes 集群中，主要服务分布在以下命名空间：

- **wallet**: eSignet, Keycloak, Inji Web, Mock Identity System, Certify
- **mimoto**: Mimoto, DataShare, MinIO
- **inji-verify**: Inji Verify

### 服务端点配置

#### 外网访问域名

| 服务 | 域名 |
|------|------|
| **Mimoto** | https://axiomid-mimoto.shrwk.com |
| **DataShare** | https://axiomid-datashare.shrwk.com |
| **Inji Web** | https://axiomid-wallet-web.shrwk.com |
| **eSignet** | https://axiomid-sigate.shrwk.com |
| **Keycloak** | https://axiomid-iam.shrwk.com/auth/realms/mosip |
| **Mock Identity** | https://axiomid-mock-mosip.shrwk.com |

#### 内部服务通信 (Kubernetes 集群内)

| 服务 | DNS 名称 |
|------|----------|
| **Keycloak** | keycloak.wallet.svc.cluster.local |
| **eSignet** | esignet.wallet.svc.cluster.local |
| **Inji Web** | inji-web.wallet.svc.cluster.local |
| **Mimoto** | mimoto.mimoto.svc.cluster.local |
| **DataShare** | datashare-service.mimoto.svc.cluster.local |

### Kubernetes 部署配置

详细的 Kubernetes 环境配置指南请参考：

- **完整配置指南**: `/home/sharework/projects/mosip/mimoto/config-management-guide-k8s.md`
- **环境配置文件**: `/home/sharework/projects/mosip/mimoto/k8s/mimoto-k8s-axiomid.properties`
- **部署脚本**: `/home/sharework/projects/mosip/scripts/deploy-mimoto-k8s.sh`

### 快速部署命令

```bash
# 使用部署脚本
cd /home/sharework/projects/mosip
./scripts/deploy-mimoto-k8s.sh

# 手动部署
kubectl apply -f mimoto/k8s/
```

### 配置验证

```bash
# 运行配置检查
./scripts/check-mimoto-config.sh

# 检查 Kubernetes 部署
kubectl get all -n mimoto

# 查看日志
kubectl logs -f deployment/mimoto -n mimoto
```

### 相关文档

- [Kubernetes 完整配置指南](./config-management-guide-k8s.md)
- [快速设置指南](./QUICK_START_CONFIG.md)
- [配置检查脚本](../../scripts/check-mimoto-config.sh)
- [Kubernetes 部署脚本](../../scripts/deploy-mimoto-k8s.sh)

