# 单点登录 SSO

> 一句话总结：SSO = 用户在身份认证中心登录一次，获得全局会话与令牌；访问其他应用时，校验令牌即可，无需重复登录。

---

## 1. SSO 定义

**单点登录（Single Sign-On，SSO）**：一种身份验证方法，允许用户**只需登录一次**，即可访问多个相关但独立的应用程序，无需为每个应用再次验证。

> **示例**：登录 Google 后，可访问 Gmail、Google Drive、YouTube 而无需再次登录。

> **小结**：
> - SSO 解决多应用重复登录问题
> - 核心是"一次登录、处处通行"
> - 与 OAuth 2.0 / OpenID Connect / SAML 协议密切相关

---

## 2. SSO 四大好处

| 好处 | 说明 |
|---|---|
| 便利性 | 用户一次登录，多应用访问 |
| 安全性 | 减少弱密码 / 重复密码风险 |
| 集中管理 | 管理员统一管控访问策略 |
| 简化 IT | 中心化管理用户访问与权限 |

> **小结**：
> - 体验 + 安全 + 管控三位一体
> - 减少密码数量 = 减少安全风险
> - 管理员从中心点统一治理

---

## 3. SSO 工作流程（14 步详解）

以访问 Google 旗下 Gmail → YouTube 为例：

| 步骤 | 动作 |
|---|---|
| 1 | 用户访问 Gmail，Gmail 发现未登录 → 重定向到 SSO 身份认证服务器 |
| 2-3 | SSO 服务器验证凭据 → 创建全局会话 + 生成令牌 |
| 4-7 | Gmail 在 SSO 服务器验证令牌 → 注册 Gmail 系统 → 返回"有效" |
| 8 | 用户从 Gmail 浏览到 YouTube |
| 9-10 | YouTube 发现未登录 → 请求 SSO 认证 → SSO 发现已登录 → 返回令牌 |
| 11-14 | YouTube 在 SSO 服务器验证令牌 → 注册 YouTube 系统 → 返回"有效" → 返回受保护资源 |

> **小结**：
> - 首次访问：重定向到 SSO → 登录 → 验证 → 通过
> - 二次访问：SSO 直接发现已登录 → 返回令牌
> - 每个应用都需向 SSO 注册"系统信息"

---

## 4. 核心角色

| 角色 | 职责 |
|---|---|
| **User** | 用户，访问多个应用 |
| **Application**（Gmail、YouTube） | 受保护资源服务器，依赖 SSO 验证 |
| **SSO Identity Provider** | 身份认证服务器，集中管理会话与令牌 |

> **小结**：
> - IdP 是 SSO 的核心中枢
> - 各应用只关心"令牌是否有效"
> - 全局会话 + 令牌是 SSO 的灵魂

---

## 5. 主流 SSO 协议对比

| 协议 | 适用场景 | 令牌格式 | 特点 |
|---|---|---|---|
| **SAML 2.0** | 企业级 SSO（SaaS） | XML | 重量级、安全性高 |
| **OAuth 2.0** | 授权（API 授权） | Access Token | 委托授权，不含身份信息 |
| **OpenID Connect** | 现代 Web/移动 SSO | ID Token（JWT） | OAuth 2.0 + 身份层 |
| **CAS** | 高校/传统企业 | Ticket | 简单、单点登录经典 |
| **Kerberos** | 内部域环境 | Ticket | Windows AD 域认证 |

> **小结**：
> - SAML = 企业 SaaS 主流
> - OIDC = 现代 Web/移动首选
> - OAuth 2.0 是授权不是认证，OIDC 才是认证
> - 选型看场景：内部域用 Kerberos，SaaS 用 SAML/OIDC

---

## 6. SSO 关键设计要点

### 6.1 全局会话管理

- SSO 服务器维护全局会话（Session）
- 用户登录成功后创建 Session，存储用户身份信息
- Session 失效时间需可配置

### 6.2 令牌机制

- **首次登录**：生成令牌返回给应用
- **后续访问**：应用携带令牌向 SSO 验证
- **令牌类型**：
  - Session ID（Cookie 携带）
  - JWT（自包含，无需存储）
  - SAML Assertion（XML 签名）

### 6.3 单点注销

- 用户在 IdP 注销 → 通知所有应用
- 应用清除本地会话
- 实现方式：
  - SLO（Single Logout）协议
  - 短令牌有效期 + 定期校验

### 6.4 跨域问题

- 浏览器同源策略限制 Cookie 共享
- 解决方案：
  - 统一顶级域名（`.example.com`）
  - Token 而非 Cookie 传递
  - 跨域 JSONP / CORS + Token

### 6.5 安全考虑

- 令牌签名验证（防伪造）
- HTTPS 加密传输
- 令牌有效期短 + 刷新机制
- 重放攻击防护（nonce / timestamp）
- CSRF 防护（SameSite Cookie）

> **小结**：
> - 全局会话 + 令牌 = SSO 核心
> - 跨域是 SSO 最大工程难点
> - 安全 = 签名 + HTTPS + 短有效期 + 防重放

---

## 7. 方案对比表

| 协议 | 复杂度 | 适用 | 移动端支持 | 推荐场景 |
|---|---|---|---|---|
| SAML 2.0 | 高 | 企业 SaaS | 中 | B2B 场景 |
| OAuth 2.0 | 中 | 授权 | 强 | API 授权 |
| **OIDC** | **中** | **现代 SSO** | **强** | **Web/移动首选** |
| CAS | 低 | 高校/传统 | 弱 | 内部系统 |
| Kerberos | 高 | Windows 域 | 弱 | 企业内网 |

| 设计要点 | 挑战 | 解决方案 |
|---|---|---|
| 全局会话 | 多应用共享状态 | IdP 集中管理 |
| 单点注销 | 通知所有应用 | SLO 协议 |
| 跨域 | Cookie 同源限制 | Token / 统一域名 |
| 安全 | 伪造 / 重放 / CSRF | 签名 + HTTPS + 短有效期 |

---

## 8. 复习清单

1. **SSO 定义？** 一次登录，多应用访问，无需重复验证。
2. **SSO 四大好处？** 便利性、安全性、集中管理、简化 IT。
3. **SSO 核心角色？** User、Application、SSO Identity Provider。
4. **SSO 首次访问流程？** 应用发现未登录 → 重定向 SSO → 验证凭据 → 创建全局会话 → 验证令牌 → 返回资源。
5. **SSO 二次访问流程？** 应用发现未登录 → SSO 发现已登录 → 返回令牌 → 验证 → 返回资源。
6. **主流 SSO 协议？** SAML 2.0、OAuth 2.0、OpenID Connect、CAS、Kerberos。
7. **OAuth 2.0 vs OIDC？** OAuth 2.0 是授权（不含身份），OIDC 在 OAuth 2.0 之上加 ID Token 实现认证。
8. **单点注销如何实现？** SLO 协议 + 短令牌有效期 + 定期校验。
9. **跨域 SSO 解决方案？** 统一顶级域名 / Token 而非 Cookie / CORS。
10. **SSO 安全防护？** 令牌签名 + HTTPS + 短有效期 + 防重放（nonce）+ CSRF 防护（SameSite）。
11. **SSO 与 OAuth 2.0 区别？** SSO 是目标（一次登录多应用），OAuth 2.0 是手段（授权框架）。
12. **选型建议？** 现代 Web/移动用 OIDC，企业 SaaS 用 SAML，内部域用 Kerberos。
