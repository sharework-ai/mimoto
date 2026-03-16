# Mimoto 配置管理指南 (Kubernetes 环境)

## 当前 Kubernetes 环境架构

### 部署架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                          │
├─────────────────────────────────────────────────────────────────┤
│  wallet 命名空间        │  mimoto 命名空间  │  inji-verify 命名空间 │
├─────────────────────────────────────────────────────────────────┤
│ • eSignet              │ • Mimoto        │ • Inji Verify         │
│ • Keycloak             │ • DataShare     │ • Verify UI           │
│ • Inji Web             │ • MinIO         │                       │
│ • Mock Identity System │                 │                       │
│ • Certify              │                 │                       │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Traefik Ingress Controller                    │
│                    (IP: 192.168.1.100)                           │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                   外部域名访问 (*.shrwk.com)                     │
└─────────────────────────────────────────────────────────────────┘
```

### 服务端点映射

#### 外网访问域名 (推荐)

| 服务 | 域名 | 用途 |
|------|------|------|
| **Mimoto** | https://axiomid-mimoto.shrwk.com | VC 钱包后端 |
| **DataShare** | https://axiomid-datashare.shrwk.com | VC 数据共享 |
| **Inji Web** | https://axiomid-wallet-web.shrwk.com | Web 钱包 |
| **eSignet** | https://axiomid-sigate.shrwk.com | 身份认证 |
| **Keycloak** | https://axiomid-iam.shrwk.com/auth/realms/mosip | IAM 服务 |
| **Mock Identity** | https://axiomid-mock-mosip.shrwk.com | 模拟身份系统 |
| **Certify** | https://axiomid-certify.shrwk.com | VC 签发 |
| **Verify** | https://axiomid-verify-service.shrwk.com | VC 验证 |

#### 内部服务通信 (Kubernetes 集群内)

| 服务 | DNS 名称 (集群内) | 端口 |
|------|------------------|------|
| **Mimoto** | mimoto.mimoto.svc.cluster.local | 8099 |
| **DataShare** | datashare-service.mimoto.svc.cluster.local | 8097 |
| **eSignet** | esignet.wallet.svc.cluster.local | 8088 |
| **Keycloak** | keycloak.wallet.svc.cluster.local | 80, 443 |
| **Inji Web** | inji-web.wallet.svc.cluster.local | 3001 |
| **Mock Identity** | mock-identity-system.wallet.svc.cluster.local | 8082 |
| **Certify** | certify-nginx.wallet.svc.cluster.local | 8091 |

#### 内网端口 (用于开发调试)

| 服务 | 本地地址 | 转发方式 |
|------|----------|----------|
| **Mimoto** | http://localhost:8099 | kubectl port-forward |
| **DataShare** | http://localhost:8097 | kubectl port-forward |
| **eSignet** | http://localhost:8088 | kubectl port-forward |
| **Keycloak** | http://localhost:8080 | kubectl port-forward |
| **Mock Identity** | http://localhost:30882 | NodePort |
| **Inji Web** | http://localhost:30765 | NodePort |

## 配置文件层次结构

Mimoto 使用 Spring Boot 的多层级配置加载机制：

```
1. mosip-config/mimoto-default.properties     # 最底层：MOSIP 平台默认配置
2. inji-config/mimoto-default.properties      # 中间层：Inji 生态默认配置
3. k8s/mimoto-k8s.properties                 # Kubernetes 环境配置
4. 环境变量                                   # 最顶层：运行时覆盖
```

## Kubernetes 环境配置方案

### 方案 1: ConfigMap + Secret (推荐用于生产)

#### 创建 Kubernetes 配置

**1. 创建 ConfigMap (非敏感配置)**

```yaml
# k8s/mimoto-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mimoto-config
  namespace: mimoto
data:
  # 服务端点配置
  mosip.api.public.url: "https://axiomid-mimoto.shrwk.com"
  mosip.inji.web.url: "https://axiomid-wallet-web.shrwk.com"
  mosip.data.share.url: "https://axiomid-datashare.shrwk.com"

  # MOSIP 服务端点
  mosip.esignet.host: "https://axiomid-sigate.shrwk.com"
  keycloak.external.url: "https://axiomid-iam.shrwk.com"
  mosip.resident.base.url: "https://axiomid-sigate.shrwk.com/resident/v1"

  # 内部服务通信
  keycloak.internal.url: "http://keycloak.wallet.svc.cluster.local"
  mosip.kernel.authmanager.url: "http://keycloak.wallet.svc.cluster.local/auth/realms/mosip"
  mosip.websub.url: "http://websub.websub"

  # VC 配置
  mosip.openid.issuers: "mimoto-issuers-config.json"
  mosip.openid.verifiers: "mimoto-trusted-verifiers.json"
  mosip.openid.htmlTemplate: "credential-template.html"

  # 缓存配置
  spring.cache.type: "redis"
  spring.data.redis.host: "redis-server"
  spring.data.redis.port: "6379"

  # 日志配置
  logging.level.io.mosip: "INFO"
  logging.level.io.mosip.mimoto: "INFO"
```

**2. 创建 Secret (敏感配置)**

```yaml
# k8s/mimoto-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mimoto-secret
  namespace: mimoto
type: Opaque
data:
  # 数据库密码 (base64 编码)
  spring.datasource.password: cG9zdGdyZXM=
  db.dbuser.password: cG9zdGdyZXM=

  # OAuth2 配置
  mosip.iam.adapter.clientsecret: your-client-secret-base64
  mosip.partner.crypto.p12.password: your-password-base64

  # Keymanager 配置
  mosip.kernel.keymanager.hsm.keystore-pass: your-keystore-password-base64
  mosip.oidc.p12.password: your-oidc-password-base64

  # Google OAuth (如果使用)
  google.oauth.client.id: your-client-id-base64
  google.oauth.client.secret: your-client-secret-base64
```

**3. 在 Deployment 中使用配置**

```yaml
# k8s/mimoto-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mimoto
  namespace: mimoto
spec:
  template:
    spec:
      containers:
      - name: mimoto
        image: mosipid/mimoto:0.20.0
        env:
        # 从 ConfigMap 加载配置
        - name: mosip_api_public_url
          valueFrom:
            configMapKeyRef:
              name: mimoto-config
              key: mosip.api.public.url

        - name: mosip_inji_web_url
          valueFrom:
            configMapKeyRef:
              name: mimoto-config
              key: mosip.inji.web.url

        # 从 Secret 加载敏感配置
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mimoto-secret
              key: spring.datasource.password

        - name: oidc_p12_password
          valueFrom:
            secretKeyRef:
              name: mimoto-secret
              key: mosip.oidc.p12.password

        # 数据库配置
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://postgres-mimoto:5432/inji_mimoto"

        - name: SPRING_DATASOURCE_USERNAME
          value: "mimotouser"

        # Redis 配置
        - name: spring_data_redis_host
          valueFrom:
            configMapKeyRef:
              name: mimoto-config
              key: spring.data.redis.host

        - name: spring_data_redis_password
          valueFrom:
            secretKeyRef:
              name: mimoto-secret
              key: spring.data.redis.password

        # 其他环境变量
        - name: active_profile_env
          value: "default,k8s"

        - name: SPRING_CONFIG_NAME
          value: "mimoto,inji"

        - name: SPRING_CONFIG_LOCATION
          value: "/home/mosip/"

        volumes:
        # 挂载配置文件
        - name: config-volume
          configMap:
            name: mimoto-files-config

        # 挂载证书文件
        - name: certs-volume
          secret:
            secretName: mimoto-certs

        volumeMounts:
        - name: config-volume
          mountPath: /home/mosip/config

        - name: certs-volume
          mountPath: /home/mosip/certs
```

### 方案 2: 环境特定配置文件

#### 创建 Kubernetes 环境配置

```bash
# mimoto-k8s.properties
cat > k8s/mimoto-k8s.properties << 'EOF'
# Mimoto Kubernetes 环境配置

# ==========================================
# 服务端点配置 - Kubernetes 环境
# ==========================================

# Mimoto 服务
mosip.api.public.url=https://axiomid-mimoto.shrwk.com
mosipbox.public.url=https://axiomid-mimoto.shrwk.com

# Inji Web
mosip.inji.web.url=https://axiomid-wallet-web.shrwk.com
mosip.inji.web.redirect.url=https://axiomid-wallet-web.shrwk.com/authorize

# DataShare
mosip.data.share.url=https://axiomid-datashare.shrwk.com
mosip.data.share.create.url=https://axiomid-datashare.shrwk.com/v1/datashare/create/static-policyid/static-subscriberid
mosip.data.share.get.url.pattern=https://axiomid-datashare.shrwk.com/v1/datashare/get/static-policyid/static-subscriberid/*

# ==========================================
# MOSIP 服务端点 - Kubernetes 内部通信
# ==========================================

# eSignet
mosip.esignet.host=https://axiomid-sigate.shrwk.com
BINDING_OTP=https://axiomid-sigate.shrwk.com/v1/esignet/binding/binding-otp
WALLET_BINDING=https://axiomid-sigate.shrwk.com/v1/esignet/binding/wallet-binding

# Keycloak (外网访问)
keycloak.external.url=https://axiomid-iam.shrwk.com
auth.server.admin.issuer.uri=https://axiomid-iam.shrwk.com/auth/realms/

# Keycloak (内部通信)
keycloak.internal.url=http://keycloak.wallet.svc.cluster.local
mosip.iam.adapter.issuerURL=http://keycloak.wallet.svc.cluster.local/auth/realms/mosip
token.request.issuerUrl=http://keycloak.wallet.svc.cluster.local/auth/realms/mosip

# Resident Service
mosip.resident.base.url=https://axiomid-sigate.shrwk.com/resident/v1
RESIDENT_OTP=${mosip.resident.base.url}/req/otp
RESIDENT_CREDENTIAL_REQUEST=${mosip.resident.base.url}/req/credential
RESIDENT_CREDENTIAL_REQUEST_STATUS=${RESIDENT_CREDENTIAL_REQUEST}/status
RESIDENT_VID=${mosip.resident.base.url}/vid

# Kernel Services (如果部署了)
# mosip.kernel.authmanager.url=http://kernel-authmanager.kernel.svc.cluster.local/
# mosip.kernel.masterdata.url=http://kernel-masterdata.kernel.svc.cluster.local
# mosip.kernel.auditmanager.url=http://kernel-auditmanager.kernel.svc.cluster.local/

# ==========================================
# WebSub 配置
# ==========================================

mosip.websub.url=http://websub.websub
mosip.event.hubUrl=${mosip.websub.url}/hub/
mosip.event.hub.subUrl=${mosip.event.hubUrl}
mosip.event.hub.pubUrl=${mosip.event.hubUrl}
mosip.event.callBackUrl=${mosip.api.public.url}/v1/mimoto/credentialshare/callback/notify

# ==========================================
# 数据库配置 - Kubernetes
# ==========================================

spring.datasource.url=jdbc:postgresql://postgres-mimoto:5432/inji_mimoto
spring.datasource.username=mimotouser
spring.datasource.password=${POSTGRES_PASSWORD}

# Keymanager 数据库
keymanager_database_url=${SPRING_DATASOURCE_URL}
keymanager_database_username=mimotouser
keymanager_database_password=${db.dbuser.password}

# ==========================================
# Redis 配置 - Kubernetes
# ==========================================

spring.session.store-type=redis
spring.data.redis.host=redis-server
spring.data.redis.port=6379
spring.data.redis.password=${REDIS_PASSWORD}
spring.data.redis.connect-timeout=10s
spring.data.redis.timeout=10s
spring.session.redis.namespace=mimoto:session:
server.servlet.session.timeout=30m

# ==========================================
# VC 配置
# ==========================================

# OpenID4VCI
mosip.openid.issuers=mimoto-issuers-config.json
mosip.openid.issuer.credentialSupported=sunbird-insurance-wellKnown.json
mosip.openid.htmlTemplate=credential-template.html

# OpenID4VP
mosip.openid.verifiers=mimoto-trusted-verifiers.json
mosip.inji.ovp.qrdata.pattern=INJI_OVP://${mosip.inji.web.url}/authorize?response_type=vp_token&resource=%s&presentation_definition=%s

# VC 下载配置
mosip.inji.vcDownloadMaxRetry=10
mosip.inji.vcDownloadPoolInterval=6000
mosip.inji.openId4VCIDownloadVCTimeout=30000

# ==========================================
# OAuth2 配置
# ==========================================

mosip.oidc.client.assertion.type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
mosip.oidc.p12.filename=oidckeystore.p12
mosip.oidc.p12.password=${oidc_p12_password}
mosip.oidc.p12.path=certs/

# Google OAuth2 (如果使用)
spring.security.oauth2.client.registration.google.client-id=${GOOGLE_OAUTH_CLIENT_ID}
spring.security.oauth2.client.registration.google.client-secret=${GOOGLE_OAUTH_CLIENT_SECRET}
spring.security.oauth2.client.registration.google.scope=profile,email
spring.security.oauth2.client.registration.google.redirect-uri={baseUrl}/oauth2/callback/{registrationId}
spring.security.oauth2.client.registration.google.authorization-grant-type=authorization_code

# ==========================================
# 安全配置
# ==========================================

# CORS
mosip.security.cors-enable=true
mosip.security.origins=https://axiomid-wallet-web.shrwk.com,https://axiomid-mimoto.shrwk.com

# CSRF
mosip.security.csrf-enable=false

# 忽略认证的端点
mosip.security.ignore-auth-urls=/safetynet/**,/actuator/**,/swagger-ui/**,/v3/api-docs/**,\
  /allProperties,/credentials/**,/credentialshare/**,/binding-otp,/wallet-binding,/get-token/**,\
  /issuers,/issuers/**,/authorize,/req/otp,/vid,/req/auth/**,/req/individualId/otp,/aid/get-individual-id,\
 /verifiers, /auth/*/token-login

# ==========================================
# Partner 配置
# ==========================================

mosip.partner.id=mpartner-default-mobile
mosip.partner.crypto.p12.filename=keystore.p12
mosip.partner.crypto.p12.password=${mosip.partner.crypto.p12.password}
mosip.partner.crypto.p12.alias=partner
mosip.partner.encryption.key=${mosip.partner.crypto.p12.password}
mosip.partner.prependThumbprint=true

mosip.datashare.partner.id=mpartner-default-resident
mosip.datashare.policy.id=mpolicy-default-resident

# Token 生成
token.request.id=io.mosip.registration.processor
token.request.appid=regproc
token.request.username=registrationprocessor
token.request.password=${token.request.password}
token.request.version=1.0
token.request.clientId=mosip-regproc-client
token.request.secretKey=${token.request.secretKey}

# ==========================================
# Keymanager 配置
# ==========================================

mosip.kernel.keymanager.hsm.config-path=/home/mosip/certs/oidckeystore.p12
mosip.kernel.keymanager.hsm.keystore-type=PKCS12
mosip.kernel.keymanager.hsm.keystore-pass=${oidc_p12_password}

mosip.kernel.keymanager.certificate.default.common-name=axiomid-mimoto.shrwk.com
mosip.kernel.keymanager.certificate.default.organizational-unit=MOSIP
mosip.kernel.keymanager.certificate.default.organization=MOSIP
mosip.kernel.keymanager.certificate.default.location=Bangalore
mosip.kernel.keymanager.certificate.default.state=KA
mosip.kernel.keymanager.certificate.default.country=IN

mosip.kernel.keymanager.autogen.appids.list=ROOT,BASE,MIMOTO
mosip.kernel.keymanager.autogen.basekeys.list=MIMOTO:user_pii

# ==========================================
# 日志配置 - Kubernetes
# ==========================================

logging.level.root=WARN
logging.level.io.mosip=INFO
logging.level.io.mosip.mimoto=INFO
logging.level.io.mosip.kernel.auth.defaultadapter=INFO
logging.level.org.springframework.http.client=INFO
logging.level.reactor.netty.http.client=INFO

# Kubernetes 日志配置
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.directory=/dev
server.tomcat.accesslog.prefix=stdout
server.tomcat.accesslog.buffered=false
server.tomcat.accesslog.suffix=
server.tomcat.accesslog.file-date-format=
server.tomcat.accesslog.pattern={"@timestamp":"%{yyyy-MM-dd'T'HH:mm:ss.SSS'Z'}t","level":"ACCESS","level_value":70000,"traceId":"%{X-B3-TraceId}i","appId":"%{X-AppId}i","statusCode":%s,"req.requestURI":"%U","bytesSent":%b,"timeTaken":%T,"appName":"${spring.application.name}"}

# ==========================================
# Actuator 配置
# ==========================================

management.endpoint.health.show-details=always
management.endpoints.web.exposure.include=info,health,refresh

# ==========================================
# 缓存配置
# ==========================================

cache.credential-issuer.wellknown.expiry-time-in-min=60
cache.issuers-config.expiry-time-in-min=60
cache.credential-issuer.authserver-wellknown.expiry-time-in-min=60
cache.pre-registered-trusted-verifiers.expiry-time-in-min=60
cache.default.expiry-time-in-min=60

# ==========================================
# 钱包配置
# ==========================================

mosip.inji.allowedAuthType=demo,otp,bio-Finger,bio-Iris,bio-Face
mosip.inji.allowedEkycAuthType=demo,otp,bio-Finger,bio-Iris,bio-Face
mosip.inji.allowedInternalAuthType=otp,bio-Finger,bio-Iris,bio-Face

# 钱包 Passcode 配置
wallet.passcode.retryBlockedUntil=60
wallet.passcode.maxFailedAttemptsAllowedPerCycle=5
wallet.passcode.maxLockCyclesAllowed=3

# QR 码配置
mosip.inji.qr.data.size.limit=4096
mosip.inji.qr.code.height=400
mosip.inji.qr.code.width=400

# ==========================================
# 认证配置
# ==========================================

mosip.iam.adapter.appid=partner
mosip.iam.adapter.clientid=mpartner-default-mobile
mosip.iam.adapter.clientsecret=${mpartner.default.mobile.secret}
mosip.iam.adapter.validate-expiry-check-rate=1440
mosip.iam.adapter.renewal-before-expiry-interval=1440
mosip.iam.adapter.self-token-renewal-enable=true
mosip.iam.adapter.disable-self-token-rest-template=true

mosip.auth.filter_disable=false
mosip.auth.adapter.impl.basepackage=io.mosip.kernel.auth.defaultadapter
mosip.kernel.auth.appids.realm.map={prereg:'mosip',ida:'mosip',registrationclient:'mosip',regproc:'mosip',partner:'mosip',resident:'mosip',admin:'mosip',crereq:'mosip',creser:'mosip',datsha:'mosip',idrepo:'mosip'}

# Wallet Binding
wallet.binding.partner.id=mpartner-default-mimotokeybinding
wallet.binding.partner.api.key=${mimoto.wallet.binding.partner.api.key}

# ==========================================
# 其他配置
# ==========================================

mosip.inji.app.id=MIMOTO
mosip.inji.audience=ida-binding
mosip.inji.issuer=residentapp
mosip.inji.warningDomainName=${mosip.api.public.url}

# 签名算法优先级
signing.algorithms.priority.order=ED25519,ES256K,ES256,RS256

# CBEFF 配置
mosip.kernel.xsdstorage-uri=https://raw.githubusercontent.com/mosip/mosip-config/develop/
mosip.kernel.xsdfile=mosip-cbeff.xsd

# Token ID 配置
mosip.kernel.tokenid.length=36

# PIN 配置
mosip.kernel.pin.length=6
EOF
```

### 方案 3: Helm Values 文件

```yaml
# helm/mimoto/values.yaml

# 镜像配置
image:
  repository: mosipid/mimoto
  tag: "0.20.0"
  pullPolicy: IfNotPresent

# 服务配置
service:
  type: ClusterIP
  port: 8099

# Ingress 配置
ingress:
  enabled: true
  className: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
  hosts:
    - host: axiomid-mimoto.shrwk.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: mimoto-tls

# 环境变量
env:
  # 基础配置
  active_profile_env: "default,k8s"
  SPRING_CONFIG_NAME: "mimoto,inji"
  SPRING_CONFIG_LOCATION: "/home/mosip/"

  # 服务端点
  mosip_api_public_url: "https://axiomid-mimoto.shrwk.com"
  mosip_inji_web_url: "https://axiomid-wallet-web.shrwk.com"
  mosip_data_share_url: "https://axiomid-datashare.shrwk.com"

  # MOSIP 服务
  mosip_esignet_host: "https://axiomid-sigate.shrwk.com"
  keycloak_external_url: "https://axiomid-iam.shrwk.com"
  keycloak_internal_url: "http://keycloak.wallet.svc.cluster.local"

  # 数据库 (从 Secret 获取)
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-mimoto:5432/inji_mimoto"
  SPRING_DATASOURCE_USERNAME: "mimotouser"
  SPRING_DATASOURCE_PASSWORD:
    secretKeyRef:
      name: mimoto-db-secret
      key: password

  # Redis
  spring_data_redis_host: "redis-server"
  spring_data_redis_port: "6379"
  spring_data_redis_password:
    secretKeyRef:
      name: mimoto-redis-secret
      key: password

  # OAuth2
  oidc_p12_password:
    secretKeyRef:
      name: mimoto-oauth-secret
      key: oidc-keystore-password

# 配置文件挂载
configFiles:
  mimoto-default.properties: |
    mosip.api.public.url=${mosip.api.public.url}
    mosip.openid.issuers=mimoto-issuers-config.json
    # ... 其他配置

  mimoto-issuers-config.json: |
    {
      "issuers": [
        {
          "issuerId": "Mosip",
          "issuerName": "MOSIP",
          "issuerUrl": "https://axiomid-sigate.shrwk.com",
          "wellKnownEndpoint": "https://axiomid-sigate.shrwk.com/v1/esignet/oauth/.well-known/openid-configuration"
        }
      ]
    }

# Secret 配置
secrets:
  database:
    password: "your-database-password"
  oauth:
    oidcKeystorePassword: "your-keystore-password"
    googleClientId: "your-google-client-id"
    googleClientSecret: "your-google-client-secret"

# 依赖服务
dependencies:
  postgresql:
    enabled: true
    host: "postgres-mimoto"
    database: "inji_mimoto"
    username: "mimotouser"

  redis:
    enabled: true
    host: "redis-server"
    port: 6379
```

## Bootstrap 配置

### mimoto-bootstrap.properties (Kubernetes)

```properties
# 配置服务器位置
# 在 Kubernetes 中，我们使用 ConfigMap 和环境变量，所以这里设置为 localhost
spring.cloud.config.uri=http://localhost/

# 要加载的配置文件列表
spring.cloud.config.name=mimoto,inji

# 应用名称
spring.application.name=mimoto

# 配置文件存储路径
config.server.file.storage.uri=http://localhost/

# 环境配置
spring.profiles.active=default,k8s

# Actuator 配置
management.endpoint.health.show-details=always
management.endpoints.web.exposure.include=info,health,refresh

# 服务配置
server.port=8099
server.servlet.context-path=/v1/mimoto
server.tomcat.max-http-response-header-size=65536

health.config.enabled=false

# OpenAPI 配置
openapi.info.title=${spring.application.name}
openapi.info.description=${spring.application.name}
openapi.info.version=1.0
openapi.info.license.name=Mosip
openapi.info.license.url=https://docs.mosip.io/platform/license
openapi.service.servers[0].url=${mosipbox.public.url}${server.servlet.context-path}
openapi.service.servers[0].description=${spring.application.name}
openapi.group.name=${openapi.info.title}
openapi.group.paths[0]=/**

springdoc.swagger-ui.disable-swagger-default-url=true
springdoc.swagger-ui.tagsSorter=alpha
springdoc.swagger-ui.operationsSorter=alpha
```

## 完整配置检查清单

### 基础服务配置
- [ ] `mosip.api.public.url=https://axiomid-mimoto.shrwk.com`
- [ ] `mosip.inji.web.url=https://axiomid-wallet-web.shrwk.com`
- [ ] `mosip.data.share.url=https://axiomid-datashare.shrwk.com`

### MOSIP 生态服务配置
- [ ] `mosip.esignet.host=https://axiomid-sigate.shrwk.com`
- [ ] `keycloak.external.url=https://axiomid-iam.shrwk.com`
- [ ] `keycloak.internal.url=http://keycloak.wallet.svc.cluster.local`

### 数据库配置
- [ ] `spring.datasource.url=jdbc:postgresql://postgres-mimoto:5432/inji_mimoto`
- [ ] `spring.datasource.username=mimotouser`
- [ ] `spring.datasource.password` (通过 Secret 设置)

### VC 配置
- [ ] `mosip.openid.issuers=mimoto-issuers-config.json`
- [ ] `mosip.openid.verifiers=mimoto-trusted-verifiers.json`

### 缓存配置
- [ ] `spring.cache.type=redis`
- [ ] `spring.data.redis.host=redis-server`

### 安全配置
- [ ] `mosip.security.cors-enable=true`
- [ ] `mosip.security.origins` 包含所有允许的域名
- [ ] OAuth2 配置完整

### 证书配置
- [ ] OIDC Keystore 配置正确
- [ ] Partner Keystore 配置正确
- [ ] Google OAuth2 配置 (如果使用)

## Kubernetes 部署示例

### 创建命名空间和配置

```bash
# 1. 创建命名空间
kubectl create namespace mimoto

# 2. 创建 ConfigMap
kubectl create configmap mimoto-config \
  --from-file=k8s/mimoto-k8s.properties \
  --from-file=k8s/mimoto-issuers-config.json \
  --from-file=k8s/mimoto-trusted-verifiers.json \
  -n mimoto

# 3. 创建 Secrets
kubectl create secret generic mimoto-secret \
  --from-literal=spring-datasource-password='your-password' \
  --from-literal=oidc-p12-password='your-keystore-password' \
  --from-literal=google-oauth-client-id='your-client-id' \
  --from-literal=google-oauth-client-secret='your-client-secret' \
  -n mimoto

# 4. 创建证书 Secret
kubectl create secret generic mimoto-certs \
  --from-file=oidckeystore.p12=path/to/your/keystore.p12 \
  -n mimoto

# 5. 部署 Mimoto
kubectl apply -f k8s/mimoto-deployment.yaml -f k8s/mimoto-service.yaml -f k8s/mimoto-ingress.yaml
```

### 验证部署

```bash
# 检查 Pod 状态
kubectl get pods -n mimoto

# 检查日志
kubectl logs -f deployment/mimoto -n mimoto

# 检查服务端点
kubectl get endpoints -n mimoto

# 端口转发测试
kubectl port-forward -n mimoto svc/mimoto 8099:8099

# 测试 API
curl http://localhost:8099/v1/mimoto/actuator/health
```

## 故障排查

### 问题 1: 服务间通信失败

**症状**: Mimoto 无法连接到其他 MOSIP 服务

**解决方案**:
1. 检查服务 DNS 名称: `kubectl get svc -A`
2. 验证网络策略: `kubectl get networkpolicies -A`
3. 测试服务连接: `kubectl exec -n mimoto -- curl http://keycloak.wallet.svc.cluster.local`

### 问题 2: Ingress 配置问题

**症状**: 外网无法访问 Mimoto 服务

**解决方案**:
1. 检查 Ingress 配置: `kubectl get ingress -n mimoto`
2. 验证 DNS 解析: `nslookup axiomid-mimoto.shrwk.com`
3. 检查 Ingress Controller 日志

### 问题 3: 配置未生效

**症状**: 修改配置后服务行为没有变化

**解决方案**:
1. 检查 ConfigMap/Secret 是否更新: `kubectl get configmap mimoto-config -n mimoto -o yaml`
2. 重启 Pod: `kubectl rollout restart deployment/mimoto -n mimoto`
3. 检查环境变量: `kubectl exec -n mimoto -- env | grep mosip`

### 问题 4: 证书问题

**症状**: OAuth2 认证失败

**解决方案**:
1. 验证 keystore 文件存在: `kubectl exec -n mimoto -- ls -la /home/mosip/certs/`
2. 检查证书密码: `kubectl get secret mimoto-secret -n mimoto -o yaml`
3. 验证证书有效性

## 监控和日志

### 日志配置

```yaml
# 使用 Fluentd/Fluent Bit 收集日志
apiVersion: v1
kind: ConfigMap
metadata:
  name: mimoto-logging
  namespace: mimoto
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/mimoto*.log
      pos_file /var/log/fluentd-mimoto.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
      </parse>
    </source>
```

### 监控指标

```yaml
# Prometheus 监控配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: mimoto-metrics
  namespace: mimoto
data:
  prometheus.yml: |
    scrape_configs:
      - job_name: 'mimoto'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - mimoto
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            regex: mimoto
            action: keep
```

## 相关资源

- [MOSIP Kubernetes 部署指南](https://docs.mosip.io/platform/deployment)
- [Spring Boot Kubernetes 配置](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html)
- [Kubernetes ConfigMap 文档](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secret 文档](https://kubernetes.io/docs/concepts/configuration/secret/)

---

**提示**: 定期检查和更新配置以适应环境变化。使用配置管理工具如 Helm 或 Kustomize 可以简化配置管理。