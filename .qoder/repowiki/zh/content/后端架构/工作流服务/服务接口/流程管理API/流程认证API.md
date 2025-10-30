# 流程认证API

<cite>
**本文档引用的文件**
- [auth.py](file://core/workflow/api/v1/flow/auth.py)
- [auth_service.py](file://core/workflow/service/auth_service.py)
- [middleware/auth.py](file://core/workflow/extensions/fastapi/middleware/auth.py)
- [casdoor.ts](file://console/frontend/src/config/casdoor.ts)
- [app_auth.py](file://core/agent/infra/app_auth.py)
- [auth_handler.lua](file://docker/astronAgent/astronRPA/volumes/nginx/lua/auth_handler.lua)
- [init_data.json](file://docker/astronAgent/casdoor/conf/init_data.json)
</cite>

## 目录
1. [简介](#简介)
2. [认证架构概述](#认证架构概述)
3. [JWT验证机制](#jwt验证机制)
4. [权限检查逻辑](#权限检查逻辑)
5. [租户隔离策略](#租户隔离策略)
6. [认证中间件工作流程](#认证中间件工作流程)
7. [API端点详细说明](#api端点详细说明)
8. [Casdoor集成](#casdoor集成)
9. [使用示例](#使用示例)
10. [错误处理与状态码](#错误处理与状态码)

## 简介
流程认证API为系统提供完整的身份验证和权限管理功能，确保只有经过授权的用户和应用才能访问和操作工作流。该系统采用多层认证机制，结合JWT令牌、HMAC签名和Casdoor单点登录，实现安全可靠的访问控制。认证系统还实现了严格的多租户隔离策略，确保不同租户之间的数据和操作完全隔离。

## 认证架构概述
流程认证系统采用分层架构设计，包含前端认证、API网关认证和后端服务认证三个层次。前端通过Casdoor实现用户身份认证，API网关通过Nginx Lua脚本进行初步认证，后端服务通过FastAPI中间件进行最终认证和权限验证。

```mermaid
graph TD
A[前端应用] --> B[Casdoor SSO]
B --> C[Nginx网关]
C --> D[认证Lua脚本]
D --> E[FastAPI中间件]
E --> F[工作流服务]
F --> G[应用管理平台]
G --> H[Redis缓存]
```

**图示来源**
- [casdoor.ts](file://console/frontend/src/config/casdoor.ts)
- [auth_handler.lua](file://docker/astronAgent/astronRPA/volumes/nginx/lua/auth_handler.lua)
- [middleware/auth.py](file://core/workflow/extensions/fastapi/middleware/auth.py)

## JWT验证机制
系统采用JWT（JSON Web Token）作为主要的认证令牌机制。JWT令牌由Casdoor签发，包含用户身份信息和权限声明。后端服务通过验证JWT签名和有效期来确认用户身份。

认证流程中，JWT令牌通过Authorization头传递，格式为"Bearer <token>"。系统支持从多个来源获取令牌，包括Authorization头、自定义Token头和Cookie中的JSESSIONID。

```mermaid
sequenceDiagram
participant 前端 as 前端应用
participant Casdoor as Casdoor SSO
participant Nginx as Nginx网关
participant 服务 as 工作流服务
前端->>Casdoor : 用户登录
Casdoor-->>前端 : 返回JWT令牌
前端->>Nginx : API请求 + Authorization头
Nginx->>Nginx : 提取JWT令牌
Nginx->>服务 : 转发请求
服务->>服务 : 验证JWT签名和有效期
服务-->>Nginx : 认证结果
Nginx-->>前端 : 响应结果
```

**图示来源**
- [casdoor.ts](file://console/frontend/src/config/casdoor.ts)
- [auth_handler.lua](file://docker/astronAgent/astronRPA/volumes/nginx/lua/auth_handler.lua)
- [middleware/auth.py](file://core/workflow/extensions/fastapi/middleware/auth.py)

## 权限检查逻辑
权限检查逻辑由auth_service.py实现，主要包含以下步骤：

1. 验证租户应用是否存在且为租户身份
2. 获取用户应用信息
3. 验证工作流是否存在
4. 检查工作流的发布状态和权限矩阵
5. 在许可证表中注册应用与工作流的绑定关系

权限检查使用TenantPublishMatrix来管理不同平台的发布权限，确保只有在正确平台上发布的工作流才能被访问。

```mermaid
flowchart TD
A[开始] --> B{路径是否需要认证?}
B --> |否| C[跳过认证]
B --> |是| D{是否存在x-consumer-username头?}
D --> |是| E[跳过认证]
D --> |否| F[获取Authorization头]
F --> G{Authorization头是否存在?}
G --> |否| H[返回401]
G --> |是| I[调用应用管理平台验证API密钥]
I --> J{验证成功?}
J --> |否| K[返回错误]
J --> |是| L[缓存应用ID]
L --> M[设置x-consumer-username头]
M --> N[继续处理请求]
```

**图示来源**
- [auth_service.py](file://core/workflow/service/auth_service.py)
- [middleware/auth.py](file://core/workflow/extensions/fastapi/middleware/auth.py)

## 租户隔离策略
系统通过多租户隔离策略确保不同租户之间的数据和操作完全隔离。租户隔离主要通过以下机制实现：

1. 租户应用标识：每个租户都有唯一的应用ID作为标识
2. 数据库隔离：通过group_id和app_id的组合实现数据隔离
3. 权限矩阵：使用TenantPublishMatrix控制不同平台的访问权限
4. 缓存隔离：在Redis中使用不同的键前缀隔离租户数据

租户隔离策略确保了即使在共享基础设施上，不同租户的数据和操作也不会相互影响。

```mermaid
classDiagram
class TenantPublishMatrix {
+Platform XINGCHEN
+Platform KAI_FANG
+Platform AI_UI
+get_publish int
+get_take_off int
+get_publish_status(int) bool
}
class License {
+app_id string
+group_id string
+created_at datetime
+updated_at datetime
}
class Flow {
+id string
+group_id string
+release_status int
+created_at datetime
+updated_at datetime
}
class App {
+id string
+source string
+is_tenant bool
+created_at datetime
+updated_at datetime
}
TenantPublishMatrix --> Flow : "控制发布状态"
App --> License : "拥有"
Flow --> License : "属于"
App --> Flow : "创建"
```

**图示来源**
- [auth_service.py](file://core/workflow/service/auth_service.py)
- [tenant_publish_matrix.py](file://core/workflow/consts/tenant_publish_matrix.py)

## 认证中间件工作流程
认证中间件(AuthMiddleware)是认证系统的核心组件，负责处理所有API请求的认证和授权。中间件的工作流程如下：

1. 检查请求路径是否需要认证
2. 检查是否存在x-consumer-username头，如果存在则跳过认证
3. 获取Authorization头并验证其存在性
4. 通过API密钥调用应用管理平台验证应用身份
5. 将验证后的应用ID添加到请求头中
6. 继续处理请求

中间件还实现了HMAC签名认证机制，通过APP_MANAGE_PLAT_KEY和APP_MANAGE_PLAT_SECRET环境变量配置的密钥对请求进行签名验证。

**本节来源**
- [middleware/auth.py](file://core/workflow/extensions/fastapi/middleware/auth.py)

## API端点详细说明
### 认证API端点
#### 发布工作流
- **HTTP方法**: POST
- **URL路径**: /api/v1/flow/publish
- **请求头**: 
  - Authorization: Bearer <token>
  - x-consumer-username: <tenant_app_id>
- **请求参数**: PublishInput对象
- **响应格式**: JSONResponse
- **状态码**:
  - 200: 发布成功
  - 401: 未授权
  - 403: 禁止访问
  - 500: 服务器错误

#### 认证绑定
- **HTTP方法**: POST
- **URL路径**: /api/v1/flow/auth
- **请求头**: 
  - Authorization: Bearer <token>
  - x-consumer-username: <tenant_app_id>
- **请求参数**: AuthInput对象
- **响应格式**: JSONResponse
- **状态码**:
  - 200: 认证成功
  - 401: 未授权
  - 403: 禁止访问
  - 500: 服务器错误

**本节来源**
- [auth.py](file://core/workflow/api/v1/flow/auth.py)

## Casdoor集成
系统通过Casdoor实现单点登录(SSO)和用户身份管理。前端应用通过Casdoor SDK与Casdoor服务器交互，完成用户登录和令牌获取。

集成主要包含以下组件：
1. 前端Casdoor SDK配置
2. 回调页面处理令牌交换
3. 令牌存储和解析
4. 登出处理

Casdoor配置包括服务器URL、客户端ID、应用名称和组织名称等，支持运行时配置和环境变量配置。

```mermaid
sequenceDiagram
participant 前端 as 前端应用
participant Casdoor as Casdoor SSO
participant 浏览器 as 浏览器存储
前端->>Casdoor : 重定向到登录页面
Casdoor-->>前端 : 返回授权码
前端->>Casdoor : 交换访问令牌
Casdoor-->>前端 : 返回JWT令牌
前端->>浏览器 : 存储访问令牌和刷新令牌
前端->>前端 : 解析用户信息
```

**图示来源**
- [casdoor.ts](file://console/frontend/src/config/casdoor.ts)
- [callback.tsx](file://console/frontend/src/pages/callback/index.tsx)

## 使用示例
### 获取访问令牌
```bash
curl -X GET "https://casdoor-server/api/login/oauth?client_id=your_client_id&redirect_uri=your_redirect_uri&response_type=code"
```

### 调用流程操作API
```bash
curl -X POST "https://api-server/api/v1/flow/publish" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "x-consumer-username: tenant_app_123" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "flow_456",
    "version": "1.0.0"
  }'
```

### 认证绑定工作流
```bash
curl -X POST "https://api-server/api/v1/flow/auth" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "x-consumer-username: tenant_app_123" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "app_789",
    "flow_id": "flow_456"
  }'
```

**本节来源**
- [casdoor.ts](file://console/frontend/src/config/casdoor.ts)
- [auth.py](file://core/workflow/api/v1/flow/auth.py)

## 错误处理与状态码
系统定义了详细的错误处理机制和状态码，确保客户端能够准确理解错误原因。

### HTTP状态码
- **200**: 成功
- **400**: 参数错误
- **401**: 未授权
- **403**: 禁止访问
- **429**: 请求过于频繁
- **500**: 服务器内部错误

### 自定义错误码
- **4001**: 缺少SESSION/Token
- **40040**: AppId认证信息查询失败
- **5000**: 内部服务器错误
- **FLOW_PUBLISH_ERROR**: 工作流发布错误
- **APP_FLOW_AUTH_BOND_ERROR**: 应用工作流认证绑定错误

错误响应格式包含code、message和sid字段，其中sid用于分布式追踪。

**本节来源**
- [auth.py](file://core/workflow/api/v1/flow/auth.py)
- [auth_handler.lua](file://docker/astronAgent/astronRPA/volumes/nginx/lua/auth_handler.lua)
- [err_code.py](file://core/workflow/exception/errors/err_code.py)