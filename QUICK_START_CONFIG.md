# Mimoto 配置快速设置指南

本指南帮助你快速设置 Mimoto 服务的配置，避免配置遗漏。

## 一分钟快速设置

### 1. 运行配置检查脚本

```bash
# 检查当前配置完整性
./scripts/check-mimoto-config.sh
```

### 2. 创建环境配置覆盖

```bash
# 复制示例配置
cd mimoto/docker-compose/config
cp mimoto-dev-overrides.properties.example mimoto-local.properties

# 根据你的环境编辑配置
vim mimoto-local.properties
```

### 3. 修改 docker-compose.yml

在 `mimoto-service` 服务中添加：

```yaml
mimoto-service:
  environment:
    # 添加环境覆盖配置
    - SPRING_CONFIG_ADDITIONAL_LOCATION=/home/mosip/config/
  volumes:
    # 添加环境覆盖配置卷
    - ./config/mimoto-local.properties:/home/mosip/config/mimoto-local.properties
```

### 4. 验证配置

```bash
# 再次运行配置检查
./scripts/check-mimoto-config.sh

# 启动服务
cd mimoto/docker-compose
docker-compose up mimoto-service
```

## 配置层次说明

```
┌─────────────────────────────────────────────────────────────┐
│  1. mosip-config/mimoto-default.properties                   │
│     (MOSIP 平台默认配置 - 不要修改)                          │
├─────────────────────────────────────────────────────────────┤
│  2. inji-config/mimoto-default.properties                    │
│     (Inji 生态默认配置 - 不要修改)                           │
├─────────────────────────────────────────────────────────────┤
│  3. mimoto/docker-compose/config/mimoto-default.properties  │
│     (Docker 环境默认配置)                                    │
├─────────────────────────────────────────────────────────────┤
│  4. mimoto/docker-compose/config/mimoto-local.properties    │
│     (你的环境覆盖配置 - 只包含需要修改的配置项)              │
├─────────────────────────────────────────────────────────────┤
│  5. 环境变量                                                │
│     (运行时最高优先级)                                       │
└─────────────────────────────────────────────────────────────┘
```

## 必需配置项检查清单

### 基础服务配置
- [ ] `mosip.api.public.url` - Mimoto 服务 URL
- [ ] `mosip.inji.web.url` - Inji Web URL
- [ ] `mosip.data.share.url` - DataShare 服务 URL

### 数据库配置
- [ ] `spring.datasource.url` - PostgreSQL 数据库 URL
- [ ] `spring.datasource.username` - 数据库用户名
- [ ] `spring.datasource.password` - 数据库密码

### VC 配置
- [ ] `mosip.openid.issuers` - Issuers 配置文件名
- [ ] `mosip.openid.verifiers` - Verifiers 配置文件名

### 安全配置（开发环境可忽略）
- [ ] `mosip.oidc.p12.password` - OIDC Keystore 密码
- [ ] OAuth2 客户端配置（如果使用）

## 常见配置场景

### 场景 1: 本地开发环境

**配置文件**: `mimoto-local.properties`

```properties
# 本地开发配置
mosip.api.public.url=http://localhost:8099
mosip.inji.web.url=http://localhost:3004
spring.datasource.url=jdbc:postgresql://localhost:5432/inji_mimoto
logging.level.io.mosip=DEBUG
```

### 场景 2: 容器化开发环境

**配置文件**: `mimoto-docker-dev.properties`

```properties
# 容器间通信
mosip.api.public.url=http://mimoto-service:8099
mosip.inji.web.url=http://inji-web:3004
spring.datasource.url=jdbc:postgresql://postgres:5432/inji_mimoto
```

### 场景 3: 生产环境

**配置文件**: `mimoto-prod.properties` + 环境变量

```properties
# 生产配置（使用环境变量传递敏感信息）
mosip.api.public.url=https://mimoto.example.com
mosip.inji.web.url=https://injiweb.example.com
logging.level.io.mosip=INFO
```

```bash
# 通过环境变量传递敏感信息
export SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
export mosip_oidc_p12_password=${KEYSTORE_PASSWORD}
```

## 配置验证

### 启动前验证

```bash
# 运行配置检查脚本
./scripts/check-mimoto-config.sh

# 验证配置文件语法
python3 -m json.tool mimoto/docker-compose/config/mimoto-issuers-config.json
```

### 运行时验证

```bash
# 检查服务启动日志
docker-compose logs mimoto-service | grep -i "config\|property"

# 访问配置端点（如果启用了 Actuator）
curl http://localhost:8099/v1/mimoto/actuator/env

# 检查特定配置值
curl http://localhost:8099/v1/mimoto/actuator/env | grep mosip.api.public.url
```

## 故障排查

### 问题 1: 配置未生效

**症状**: 修改了配置文件，但服务行为没有变化

**解决方案**:
1. 检查配置文件路径是否正确
2. 确认配置文件在容器内的挂载位置
3. 检查配置项名称是否正确
4. 查看服务启动日志

### 问题 2: 配置冲突

**症状**: 不同配置文件中的同一配置项冲突

**解决方案**:
1. 使用配置检查脚本查看配置优先级
2. 在环境覆盖文件中明确指定值
3. 使用环境变量覆盖

### 问题 3: JSON 配置文件格式错误

**症状**: Issuers 或 Verifiers 配置无法加载

**解决方案**:
```bash
# 验证 JSON 格式
python3 -m json.tool config/mimoto-issuers-config.json
python3 -m json.tool config/mimoto-trusted-verifiers.json
```

## 最佳实践

### 1. 保持默认配置完整

```bash
# ✅ 好的做法 - 创建覆盖文件
cat > config/mimoto-local.properties << EOF
mosip.api.public.url=http://localhost:8099
EOF

# ❌ 不好的做法 - 直接修改默认配置
sed -i 's|mosip.api.public.url=.*|mosip.api.public.url=http://localhost:8099|' \
  config/mimoto-default.properties
```

### 2. 敏感信息管理

```bash
# ✅ 使用环境变量
export SPRING_DATASOURCE_PASSWORD="secure-password"

# ❌ 不要在配置文件中硬编码
# echo "spring.datasource.password=secure-password" >> config/mimoto-local.properties
```

### 3. 配置版本控制

```bash
# .gitignore 示例
mimoto-local.properties
mimoto-prod.properties
*.p12
*.jks
```

### 4. 配置文档化

```properties
# 在配置文件中添加注释
# 配置项说明
# 修改日期: 2024-01-01
# 修改原因: 本地开发环境配置
mosip.api.public.url=http://localhost:8099
```

## 相关资源

- [完整配置管理指南](./config-management-guide.md)
- [MOSIP 配置参考](https://docs.mosip.io/)
- [Spring Boot 配置文档](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html)

## 获取帮助

如果遇到配置问题：

1. 运行配置检查脚本: `./scripts/check-mimoto-config.sh`
2. 查看服务日志: `docker-compose logs mimoto-service`
3. 参考完整配置管理指南
4. 检查 MOSIP 官方文档

---

**提示**: 配置检查脚本会验证所有必需的配置项，建议在每次修改配置后运行。
