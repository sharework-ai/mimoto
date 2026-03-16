# Mimoto 依赖 Resident Service 的功能详解

## 📋 概述

Mimoto 作为 VC 钱包后端服务，**多个核心功能依赖于 Resident Service**。Resident Service 是 MOSIP 平台中的一个独立服务，负责处理居民的各类操作请求。

## 🎯 依赖关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Mimoto (钱包后端)                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────────────┐
        │     Resident Service (居民服务)             │
        │  https://api.collab.mosip.net/resident/v1   │
        └─────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────────────┐
        │      MOSIP 平台核心功能                     │
        │  • 身份验证                                │
        │  • 凭证签发                                │
        │  • VID 管理                                │
        └─────────────────────────────────────────────┘
```

## 🔗 功能依赖详解

### 1. 📱 VC 凭证下载功能

**依赖的 Resident Service 端点:**
- `POST /resident/v1/req/credential` - 请求凭证下载
- `GET /resident/v1/req/credential/status/{requestId}` - 查询凭证下载状态

**Mimoto 控制器:**
```java
// CredentialShareController.java
restClientService.postApi(ApiName.RESIDENT_CREDENTIAL_REQUEST, "", "",
    mosipCredentialRequestPayload, CredentialRequestResponseDTO.class);

restClientService.getApi(ApiName.RESIDENT_CREDENTIAL_REQUEST_STATUS, pathSegment, "", "",
    ResponseWrapper.class);
```

**使用场景:**
1. **凭证请求** - 用户在钱包中请求下载 VC
2. **状态查询** - 轮询查询凭证签发状态
3. **WebSub 回调** - 接收凭证签发完成的通知

**API 端点:**
```
POST /v1/mimoto/credentialshare/request
GET /v1/mimoto/credentialshare/{requestId}/status
```

### 2. 🔐 OTP 认证功能

**依赖的 Resident Service 端点:**
- `POST /resident/v1/req/otp` - 请求 OTP

**Mimoto 控制器:**
```java
// ResidentServiceController.java
restClientService.postApi(ApiName.RESIDENT_OTP, "", "",
    mosipOTPRequestPayload, ResponseWrapper.class);
```

**使用场景:**
1. **凭证下载 OTP** - 下载 VC 前需要 OTP 验证
2. **身份验证 OTP** - 敏感操作前的身份验证
3. **通知渠道选择** - 支持短信/邮件/邮箱 OTP

**API 端点:**
```
POST /v1/mimoto/req/otp
```

### 3. 🆔 VID 生成功能

**依赖的 Resident Service 端点:**
- `POST /resident/v1/vid` - 生成虚拟 ID

**Mimoto 控制器:**
```java
// ResidentServiceController.java
restClientService.postApi(ApiName.RESIDENT_VID, "", "",
    mosipVIDRequestPayload, ResponseWrapper.class);
```

**使用场景:**
1. **临时身份** - 为用户生成临时虚拟身份
2. **隐私保护** - 使用 VID 替代真实 UIN
3. **凭证请求** - 某些凭证需要 VID 而非 UIN

**API 端点:**
```
POST /v1/mimoto/vid
```

### 4. 🔒 认证锁定/解锁功能

**依赖的 Resident Service 端点:**
- `POST /resident/v1/req/auth/lock` - 锁定认证
- `POST /resident/v1/req/auth/unlock` - 解锁认证

**Mimoto 控制器:**
```java
// ResidentServiceController.java
restClientService.postApi(ApiName.RESIDENT_AUTH_LOCK, "", "",
    mosipAuthLockRequestPayload, ResponseWrapper.class);

restClientService.postApi(ApiName.RESIDENT_AUTH_UNLOCK, "", "",
    mosipAuthUnlockRequestPayload, ResponseWrapper.class);
```

**使用场景:**
1. **安全保护** - 检测到异常时锁定用户认证
2. **自助解锁** - 用户通过 OTP 解锁认证
3. **生物识别锁定** - 生物识别失败次数过多时锁定

**API 端点:**
```
POST /v1/mimoto/req/auth/lock
POST /v1/mimoto/req/auth/unlock
```

### 5. 🆔 AID (Application ID) 相关功能

**依赖的 Resident Service 端点:**
- `POST /resident/v1/individualId/otp` - 请求 AID OTP
- `POST /resident/v1/aid/status` - 查询 AID 状态

**Mimoto 控制器:**
```java
// ResidentServiceController.java
restClientService.postApi(ApiName.RESIDENT_INDIVIDUALID_OTP, "", "",
    mosipOTPRequestPayload, OTPResponseDTO.class);

restClientService.postApi(ApiName.RESIDENT_AID_GET_INDIVIDUALID, "", "",
    mosipAuthLockRequestPayload, ResponseWrapper.class);
```

**使用场景:**
1. **AID 转换** - 将 Application ID 转换为真实身份 ID
2. **状态查询** - 查询 AID 生成/处理状态
3. **隐私保护** - 使用 AID 隐藏真实身份

**API 端点:**
```
POST /v1/mimoto/req/individualId/otp
POST /v1/mimoto/aid/get-individual-id
```

## 📊 功能依赖矩阵

| Mimoto 功能 | 依赖的 Resident Service 端点 | 重要性 | 影响 |
|------------|------------------------------|--------|------|
| **VC 凭证下载** | `/req/credential` | 🔴 **关键** | 完全无法下载 VC |
| **凭证状态查询** | `/req/credential/status` | 🔴 **关键** | 无法查询下载状态 |
| **OTP 验证** | `/req/otp` | 🔴 **关键** | 无法进行身份验证 |
| **VID 生成** | `/vid` | 🟡 **重要** | 无法生成临时身份 |
| **认证锁定** | `/req/auth/lock` | 🟢 **可选** | 无法锁定认证 |
| **认证解锁** | `/req/auth/unlock` | 🟢 **可选** | 无法解锁认证 |
| **AID OTP** | `/individualId/otp` | 🟡 **重要** | 无法处理 AID |
| **AID 状态** | `/aid/status` | 🟡 **重要** | 无法查询 AID |

## 🔴 关键依赖功能

### 1. VC 凭证下载流程

```
用户请求 VC
    ↓
Mimoto: /credentialshare/request
    ↓
Resident Service: /req/credential
    ↓
MOSIP 平台签发 VC
    ↓
WebSub 通知
    ↓
Mimoto: /credentialshare/callback/notify
    ↓
用户下载 VC
```

**如果 Resident Service 不可用:**
- ❌ 无法请求 VC 签发
- ❌ 无法查询签发状态
- ❌ VC 下载功能完全失效

### 2. OTP 认证流程

```
用户发起需要验证的操作
    ↓
Mimoto: /req/otp
    ↓
Resident Service: /req/otp
    ↓
发送 OTP 到用户
    ↓
用户输入 OTP
    ↓
验证通过，继续操作
```

**如果 Resident Service 不可用:**
- ❌ 无法获取 OTP
- ❌ 无法进行身份验证
- ❌ 所有需要 OTP 的功能失效

## 🟡 重要依赖功能

### 1. VID 生成

**使用场景:**
- 用户希望使用临时身份
- 凭证请求需要 VID 而非 UIN
- 隐私保护需求

**如果 Resident Service 不可用:**
- ❌ 无法生成新的 VID
- ⚠️ 可以使用现有的 VID/UIN
- ⚠️ 部分功能受限但不是完全失效

### 2. AID 相关功能

**使用场景:**
- 处理基于 Application ID 的请求
- 转换临时身份到真实身份
- 查询身份处理状态

**如果 Resident Service 不可用:**
- ❌ 无法处理 AID 相关请求
- ⚠️ 需要直接使用 UIN/VID
- ⚠️ 影响隐私保护功能

## 🟢 可选依赖功能

### 1. 认证锁定/解锁

**使用场景:**
- 检测到异常行为时锁定用户
- 用户自助解锁认证
- 生物识别失败次数限制

**如果 Resident Service 不可用:**
- ⚠️ 无法主动锁定/解锁
- ⚠️ 可以依赖其他安全机制
- ✅ 核心功能不受影响

## 🚨 Resident Service 不可用的影响

### 完全失效的功能
- ✗ VC 凭证下载
- ✗ 凭证状态查询
- ✗ OTP 身份验证
- ✗ AID 相关功能

### 部分受限的功能
- ⚠️ VID 生成（可使用现有 VID）
- ⚠️ 认证锁定/解锁（可依赖其他机制）

### 不受影响的功能
- ✅ VC 展示 (已下载的 VC)
- ✅ VC 验证
- ✅ Issuers 管理
- ✅ Verifiers 管理
- ✅ 钱包基础功能

## 🔧 配置检查

### 检查 Resident Service 配置

```bash
# 检查配置
grep "mosip.resident.base.url" /path/to/mimoto.properties

# 测试连接
curl https://api.collab.mosip.net/resident/v1/actuator/health

# 测试端点
curl -X POST https://api.collab.mosip.net/resident/v1/req/otp \
  -H "Content-Type: application/json" \
  -d '{"individualId":"test","otpChannel":"EMAIL"}'
```

### 常见配置错误

❌ **错误配置:**
```properties
# 错误：指向 eSignet
mosip.resident.base.url=https://axiomid-sigate.shrwk.com/resident/v1
```

✅ **正确配置:**
```properties
# 正确：指向 Resident Service
mosip.resident.base.url=https://api.collab.mosip.net/resident/v1
```

## 📝 总结

**Mimoto 对 Resident Service 的依赖程度：🔴 高度依赖**

### 核心依赖 (不可用 = 功能失效)
1. VC 凭证下载
2. OTP 身份验证
3. 凭证状态查询

### 重要依赖 (不可用 = 功能受限)
1. VID 生成
2. AID 相关功能

### 可选依赖 (不可用 = 有替代方案)
1. 认证锁定/解锁

**配置优先级：🔴 必须正确配置 Resident Service**

如果 Resident Service 配置错误或不可用，Mimoto 的核心 VC 钱包功能将无法正常工作！
