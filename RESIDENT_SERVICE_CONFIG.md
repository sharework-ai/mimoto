# Resident Service 配置说明

## 问题分析

在 MOSIP 架构中，**Resident Service** 是一个独立的服务，负责处理居民的 OTP、凭证请求、VID 生成等功能。

### 常见配置误区

❌ **错误配置**:
```properties
# 错误：指向 eSignet
mosip.resident.base.url=https://axiomid-sigate.shrwk.com/resident/v1
```

❌ **错误配置**:
```properties
# 错误：指向 mock-identity-system
mosip.resident.base.url=https://axiomid-mock-mosip.shrwk.com/resident/v1
```

### 正确配置

✅ **正确配置**:
```properties
# 正确：使用外部 MOSIP 协作环境的 Resident Service
mosip.resident.base.url=https://api.collab.mosip.net/resident/v1
```

## MOSIP 服务架构

### 服务职责分工

| 服务 | 职责 | 是否包含 Resident Service |
|------|------|--------------------------|
| **Resident Service** | 处理居民相关操作（OTP、凭证、VID等） | ✅ **这是专门的 Resident Service** |
| **eSignet** | OAuth 2.0/OIDC 身份提供者 | ❌ 不包含 |
| **Mock Identity System** | 模拟身份系统用于测试 | ❌ 不包含 |
| **Mimoto** | VC 钱包后端服务 | ❌ 不包含（但会调用 Resident Service） |
| **Inji Certify** | VC 签发服务 | ❌ 不包含 |

### 服务调用关系

```
┌─────────────────────────────────────────────────────────────────┐
│                      MOSIP 服务调用链                          │
└─────────────────────────────────────────────────────────────────┘

Mimoto (钱包)
    ↓ 调用
Resident Service (居民服务)
    ↓ 提供接口
OTP/凭证/VID 等功能

eSignet (身份认证)
    ↓ 独立服务
OAuth 2.0/OIDC 认证

Mock Identity System (模拟身份)
    ↓ 独立服务
测试用身份数据
```

## 配置验证

### 验证 Resident Service 可用性

```bash
# 测试外部 MOSIP 协作环境的 Resident Service
curl https://api.collab.mosip.net/resident/v1/actuator/health

# 应该返回健康状态
# {"status":"UP","details":{...}}
```

### 验证配置文件

```bash
# 检查当前配置
grep "mosip.resident.base.url" /home/sharework/projects/mosip/mimoto/k8s/mimoto-k8s-axiomid.properties

# 应该显示:
# mosip.resident.base.url=https://api.collab.mosip.net/resident/v1
```

## 完整的 Mimoto Resident Service 配置

```properties
# ==========================================
# Resident Service 配置
# ==========================================

# Resident Service 基础 URL
# 使用外部 MOSIP 协作环境的 Resident Service
mosip.resident.base.url=https://api.collab.mosip.net/resident/v1

# Resident Service 具体端点
RESIDENT_OTP=${mosip.resident.base.url}/req/otp
RESIDENT_CREDENTIAL_REQUEST=${mosip.resident.base.url}/req/credential
RESIDENT_CREDENTIAL_REQUEST_STATUS=${RESIDENT_CREDENTIAL_REQUEST}/status
RESIDENT_VID=${mosip.resident.base.url}/vid
RESIDENT_AUTH_LOCK=${mosip.resident.base.url}/req/auth-lock
RESIDENT_AUTH_UNLOCK=${mosip.resident.base.url}/req/auth-unlock
RESIDENT_INDIVIDUALID_OTP=${mosip.resident.base.url}/individualId/otp
RESIDENT_AID_GET_INDIVIDUALID=${mosip.resident.base.url}/aid/status
```

## 不同环境的配置策略

### 开发/测试环境

```properties
# 使用外部 MOSIP 协作环境
mosip.resident.base.url=https://api.collab.mosip.net/resident/v1
```

### 生产环境

```properties
# 部署本地 Resident Service 或使用生产环境 URL
mosip.resident.base.url=https://prod-mosip.example.com/resident/v1
```

### 混合环境

```properties
# 本地服务 + 外部 Resident Service
mosip.resident.base.url=https://api.collab.mosip.net/resident/v1
mosip.esignet.host=https://axiomid-sigate.shrwk.com
keycloak.external.url=https://axiomid-iam.shrwk.com
```

## 故障排查

### 问题 1: Resident Service 端点 404

**症状**: 调用 Resident Service 端点返回 404

**原因**: 配置了错误的服务 URL

**解决方案**:
```bash
# 检查配置
grep "mosip.resident.base.url" /path/to/mimoto.properties

# 修改为正确的 URL
mosip.resident.base.url=https://api.collab.mosip.net/resident/v1
```

### 问题 2: 跨域访问问题

**症状**: 浏览器调用 Resident Service 时 CORS 错误

**解决方案**: 在 Resident Service 端配置 CORS，或通过后端代理

### 问题 3: 认证失败

**症状**: 调用 Resident Service 返回 401/403

**解决方案**: 检查认证配置和密钥

## 相关文档

- [MOSIP Resident Service API 文档](https://api.collab.mosip.net/resident/v1/swagger-ui/index.html)
- [Mimoto 配置管理指南](./config-management-guide-k8s.md)
- [配置验证脚本](../../scripts/verify-resident-service-config.sh)

## 总结

**关键点**:
1. ❌ eSignet **不是** Resident Service
2. ❌ Mock Identity System **不是** Resident Service
3. ✅ Resident Service 是**独立的服务**
4. ✅ 在当前环境中，应使用 `https://api.collab.mosip.net/resident/v1`

**正确的服务架构**:
- **Resident Service**: 处理居民操作
- **eSignet**: 处理 OAuth/OIDC 认证
- **Mimoto**: 钱包服务，调用其他服务
- **Mock Identity**: 测试用模拟数据

通过正确的服务配置，确保 Mimoto 能够正常调用 Resident Service 的各项功能。
