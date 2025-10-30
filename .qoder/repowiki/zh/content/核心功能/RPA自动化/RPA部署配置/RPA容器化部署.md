# RPA容器化部署详细文档

<cite>
**本文档中引用的文件**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml)
- [Dockerfile](file://core/plugin/rpa/Dockerfile)
- [main.py](file://core/plugin/rpa/main.py)
- [pyproject.toml](file://core/plugin/rpa/pyproject.toml)
- [app.py](file://core/plugin/rpa/api/app.py)
- [config.env](file://docker/astronAgent/config/rpa/config.env)
- [docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)
- [nginx.conf](file://docker/astronAgent/nginx/nginx.conf)
</cite>

## 目录
1. [概述](#概述)
2. [系统架构](#系统架构)
3. [Docker Compose 配置详解](#docker-compose-配置详解)
4. [RPA服务容器化实现](#rpa服务容器化实现)
5. [多阶段构建策略](#多阶段构建策略)
6. [服务启动与初始化](#服务启动与初始化)
7. [健康检查配置](#健康检查配置)
8. [环境变量管理](#环境变量管理)
9. [负载均衡与反向代理](#负载均衡与反向代理)
10. [多实例部署配置](#多实例部署配置)
11. [监控与可观测性](#监控与可观测性)
12. [最佳实践与优化建议](#最佳实践与优化建议)

## 概述

本文档详细介绍基于Docker和docker-compose的RPA（机器人流程自动化）服务容器化部署方案。该方案采用微服务架构，支持高可用、可扩展的RPA服务部署，具备完善的监控、日志管理和负载均衡能力。

### 核心特性

- **模块化设计**：基于FastAPI框架的微服务架构
- **容器化部署**：使用Docker进行标准化部署
- **健康监控**：内置健康检查机制
- **负载均衡**：支持多实例部署
- **可观测性**：集成OpenTelemetry监控体系
- **环境隔离**：独立的配置管理和环境变量

## 系统架构

```mermaid
graph TB
subgraph "客户端层"
WebUI[Web界面]
API[API客户端]
end
subgraph "负载均衡层"
Nginx[Nginx反向代理]
end
subgraph "RPA服务层"
RPA1[RPA服务实例1]
RPA2[RPA服务实例2]
RPA3[RPA服务实例N]
end
subgraph "基础设施层"
MySQL[(MySQL数据库)]
Redis[(Redis缓存)]
MinIO[(对象存储)]
RabbitMQ[(消息队列)]
end
subgraph "监控层"
OTLP[OpenTelemetry收集器]
Prometheus[指标监控]
Grafana[可视化面板]
end
WebUI --> Nginx
API --> Nginx
Nginx --> RPA1
Nginx --> RPA2
Nginx --> RPA3
RPA1 --> MySQL
RPA1 --> Redis
RPA1 --> MinIO
RPA1 --> RabbitMQ
RPA2 --> MySQL
RPA2 --> Redis
RPA2 --> MinIO
RPA2 --> RabbitMQ
RPA3 --> MySQL
RPA3 --> Redis
RPA3 --> MinIO
RPA3 --> RabbitMQ
RPA1 --> OTLP
RPA2 --> OTLP
RPA3 --> OTLP
OTLP --> Prometheus
Prometheus --> Grafana
```

**图表来源**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml#L1-L260)
- [nginx.conf](file://docker/astronAgent/nginx/nginx.conf#L1-L150)

## Docker Compose 配置详解

### 服务定义结构

docker-compose.yml文件定义了完整的RPA服务生态系统，包含以下核心组件：

#### 数据库服务配置

```mermaid
graph LR
subgraph "数据库集群"
MySQL[MySQL 8.4.6<br/>端口: 3306<br/>数据持久化]
Redis[Redis Bitnami<br/>端口: 6379<br/>缓存集群]
MinIO[MinIO对象存储<br/>端口: 9000/9001<br/>文件存储]
end
subgraph "数据卷管理"
MySQLVol[MySQL数据卷<br/>./volumes/mysql]
RedisVol[Redis数据卷<br/>./data/bitnami/redis]
MinIOVol[MinIO数据卷<br/>./data/minio]
end
MySQL --> MySQLVol
Redis --> RedisVol
MinIO --> MinIOVol
```

**图表来源**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml#L3-L100)

#### 关键服务配置参数

| 服务名称 | 镜像版本 | 端口映射 | 健康检查间隔 | 资源限制 |
|---------|---------|---------|-------------|----------|
| rpa-mysql | mysql:8.4.6 | 3306:3306 | 10s | CPU: 2核, 内存: 4GB |
| rpa-redis | bitnami/redis:latest | 6379:6379 | 5s | CPU: 1核, 内存: 2GB |
| rpa-minio | minio/minio:RELEASE.2025-06-13T11-33-47Z-cpuv1 | 9000:9000, 9001:9001 | 30s | CPU: 1核, 内存: 2GB |
| openresty-nginx | openresty/openresty:1.27.1.1-alpine | 80:32742 | 30s | CPU: 1核, 内存: 1GB |

**节来源**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml#L3-L260)

### 卷挂载策略

系统采用分层卷挂载策略，确保数据持久化和配置灵活性：

- **配置文件卷**：`./volumes/nginx/default.conf` → `/etc/nginx/conf.d/default.conf`
- **日志目录卷**：`./logs/nginx` → `/usr/local/openresty/nginx/logs`
- **应用程序卷**：`../backend/ai-service/app` → `/app/app`
- **本地数据卷**：`./data/mysql` → `/var/lib/mysql`

### 环境变量注入

通过.env文件统一管理环境变量，支持动态配置：

```yaml
env_file: *env_file
environment:
  - DATABASE_URL=mysql+aiomysql://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@rpa-mysql:3306/${DATABASE_NAME}
  - REDIS_URL=redis://rpa-redis:6379/${REDIS_DB}
  - AICHAT_BASE_URL=${AICHAT_BASE_URL}
  - AICHAT_API_KEY=${AICHAT_API_KEY}
```

**节来源**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml#L120-L140)

## RPA服务容器化实现

### 主要服务组件

```mermaid
classDiagram
class RPAServer {
+start() void
+setup_server() void
+load_polaris() void
+check_env() void
+set_config() void
+start_uvicorn() void
}
class FastAPIApp {
+include_router() void
+setup_middlewares() void
+configure_logging() void
}
class UvicornServer {
+Config config
+Server server
+run() void
+reload : bool
+workers : int
}
class HealthCheck {
+check_database() bool
+check_redis() bool
+check_minio() bool
+check_dependencies() bool
}
RPAServer --> FastAPIApp : 创建
FastAPIApp --> UvicornServer : 启动
RPAServer --> HealthCheck : 使用
```

**图表来源**
- [app.py](file://core/plugin/rpa/api/app.py#L1-L161)
- [main.py](file://core/plugin/rpa/main.py#L1-L112)

### 容器启动流程

RPA服务的启动遵循严格的初始化顺序：

1. **Python路径设置**：自动配置PYTHONPATH以支持模块导入
2. **环境变量加载**：从config.env文件加载配置参数
3. **服务初始化**：启动必要的基础设施服务
4. **配置验证**：检查所有必需的环境变量
5. **日志配置**：设置日志级别和输出路径
6. **Uvicorn启动**：启动ASGI应用服务器

**节来源**
- [main.py](file://core/plugin/rpa/main.py#L1-L112)

## 多阶段构建策略

### Dockerfile 架构

```mermaid
flowchart TD
Start([开始构建]) --> BaseImage[基础镜像<br/>python:3.11-slim]
BaseImage --> InstallUV[安装UV包管理器]
InstallUV --> CopyLock[复制依赖锁定文件<br/>pyproject.toml & uv.lock]
CopyLock --> InstallDeps[安装Python依赖<br/>uv sync]
InstallDeps --> CopyCommon[复制通用模块<br/>core/common]
CopyCommon --> CopyRPA[复制RPA模块<br/>core/plugin/rpa]
CopyRPA --> SetCMD[设置启动命令<br/>uv run main.py]
SetCMD --> End([构建完成])
```

**图表来源**
- [Dockerfile](file://core/plugin/rpa/Dockerfile#L1-L19)

### 依赖管理策略

系统采用uv包管理器进行依赖管理，具有以下优势：

- **快速安装**：比pip更快的包安装速度
- **确定性构建**：通过uv.lock确保依赖一致性
- **国内镜像源**：使用清华大学开源软件镜像站
- **缓存优化**：禁用缓存以确保每次构建的一致性

**节来源**
- [Dockerfile](file://core/plugin/rpa/Dockerfile#L1-L19)
- [pyproject.toml](file://core/plugin/rpa/pyproject.toml#L1-L45)

### 镜像优化技术

- **多阶段构建**：减少最终镜像大小
- **基础镜像选择**：使用python:3.11-slim最小化体积
- **环境变量配置**：优化运行时性能
- **工作目录设置**：合理组织文件结构

## 服务启动与初始化

### 初始化流程详解

```mermaid
sequenceDiagram
participant Main as main.py
participant Setup as setup_python_path()
participant Env as load_env_file()
participant Init as initialize_services()
participant Uvicorn as start_uvicorn()
Main->>Setup : 设置Python路径
Setup->>Setup : 添加项目根目录到PYTHONPATH
Setup-->>Main : 路径配置完成
Main->>Env : 加载环境变量
Env->>Env : 解析config.env文件
Env->>Env : 设置环境变量
Env-->>Main : 配置加载完成
Main->>Init : 初始化服务
Init->>Init : 启动设置服务
Init->>Init : 启动日志服务
Init->>Init : 启动OTLP服务
Init->>Init : 启动Kafka服务
Init-->>Main : 服务初始化完成
Main->>Uvicorn : 启动Uvicorn服务器
Uvicorn->>Uvicorn : 创建FastAPI应用
Uvicorn->>Uvicorn : 绑定端口8080
Uvicorn-->>Main : 服务器启动成功
```

**图表来源**
- [main.py](file://core/plugin/rpa/main.py#L15-L112)
- [app.py](file://core/plugin/rpa/api/app.py#L30-L160)

### FastAPI应用工厂模式

系统采用工厂模式创建FastAPI应用实例：

```python
def rpa_server_app() -> FastAPI:
    app = FastAPI()
    app.include_router(router)
    return app
```

这种设计模式提供了：
- **模块化路由**：清晰的API路由分离
- **中间件配置**：灵活的请求处理管道
- **生命周期管理**：优雅的应用启动和关闭
- **测试友好**：便于单元测试和集成测试

**节来源**
- [app.py](file://core/plugin/rpa/api/app.py#L140-L161)

### Uvicorn配置优化

```python
uvicorn_config = uvicorn.Config(
    app=rpa_server_app(),
    host="0.0.0.0",
    port=int(os.getenv(const.SERVICE_PORT_KEY, "17198")),
    workers=20,
    reload=False,
    log_config=None,
)
```

关键配置参数说明：
- **主机绑定**：监听所有网络接口
- **端口配置**：支持环境变量动态配置
- **工作进程数**：根据CPU核心数调整
- **热重载**：生产环境禁用以提高性能

**节来源**
- [app.py](file://core/plugin/rpa/api/app.py#L120-L140)

## 健康检查配置

### 多层次健康检查

系统实现了多层次的健康检查机制：

```mermaid
graph TD
subgraph "容器级健康检查"
ContainerHealth[容器健康状态]
ContainerHealth --> Ready[就绪检查]
ContainerHealth --> Live[存活检查]
end
subgraph "服务级健康检查"
ServiceHealth[服务健康状态]
ServiceHealth --> DBCheck[数据库连接]
ServiceHealth --> RedisCheck[Redis连接]
ServiceHealth --> MinIOCheck[MinIO连接]
ServiceHealth --> DepCheck[依赖检查]
end
subgraph "应用级健康检查"
AppHealth[应用健康状态]
AppHealth --> APICheck[API响应]
AppHealth --> TaskCheck[任务处理]
AppHealth --> ConfigCheck[配置验证]
end
ContainerHealth --> ServiceHealth
ServiceHealth --> AppHealth
```

**图表来源**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml#L20-L50)

### 具体健康检查配置

#### MySQL健康检查
```yaml
healthcheck:
  test:
    - CMD
    - mysqladmin
    - ping
    - -h
    - localhost
    - -u${DATABASE_USERNAME}
    - -p${DATABASE_PASSWORD}
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s
```

#### Redis健康检查
```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 10s
  retries: 10
  start_period: 10s
```

#### MinIO健康检查
```yaml
healthcheck:
  test:
    - CMD-SHELL
    - '/usr/bin/mc alias set health_check http://localhost:9000 ${MINIO_AK} ${MINIO_SK} && /usr/bin/mc ready health_check'
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s
```

**节来源**
- [docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml#L20-L100)

### 自定义健康检查端点

在FastAPI应用中，可以添加自定义的健康检查端点：

```python
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow(),
        "dependencies": {
            "database": await check_database_health(),
            "redis": await check_redis_health(),
            "minio": await check_minio_health()
        }
    }
```

## 环境变量管理

### 配置文件结构

RPA服务的环境变量配置采用分层管理模式：

```mermaid
graph TB
subgraph "配置层次"
GlobalEnv[全局环境变量<br/>.env文件]
ServiceEnv[服务特定配置<br/>config.env]
RuntimeEnv[运行时参数<br/>Docker环境]
end
subgraph "配置类别"
ServiceConf[服务配置<br/>SERVICE_PORT, SERVICE_NAME]
DBConf[数据库配置<br/>DATABASE_URL, DATABASE_NAME]
LogConf[日志配置<br/>LOG_LEVEL, LOG_PATH]
OTLPConf[监控配置<br/>OTLP_ENDPOINT, OTLP_ENABLE]
ExtConf[外部服务配置<br/>XIAOWU_RPA_*]
end
GlobalEnv --> ServiceConf
ServiceEnv --> DBConf
ServiceEnv --> LogConf
ServiceEnv --> OTLPConf
ServiceEnv --> ExtConf
RuntimeEnv --> ServiceConf
RuntimeEnv --> DBConf
```

**图表来源**
- [config.env](file://docker/astronAgent/config/rpa/config.env#L1-L80)

### 关键配置参数

#### 服务基本信息
- `SERVICE_SUB`: 服务子标识符（rpa）
- `SERVICE_NAME`: 服务名称（RPA）
- `SERVICE_LOCATION`: 服务位置（hf）
- `SERVICE_PORT`: 服务端口（17198）

#### 日志配置
- `LOG_LEVEL`: 日志级别（INFO）
- `LOG_PATH`: 日志路径（logs）

#### OpenTelemetry监控
- `OTLP_ENDPOINT`: 监控端点地址
- `OTLP_SERVICE_NAME`: 服务名称标识
- `OTLP_ENABLE`: 监控启用标志（1=启用）

#### 外部服务集成
- `XIAOWU_RPA_TASK_CREATE_URL`: 任务创建URL
- `XIAOWU_RPA_TASK_QUERY_URL`: 任务查询URL
- `XIAOWU_RPA_TIMEOUT`: 超时时间（3000ms）

**节来源**
- [config.env](file://docker/astronAgent/config/rpa/config.env#L1-L80)

### 环境变量验证机制

系统在启动时会验证所有必需的环境变量：

```python
def check_env() -> None:
    required_keys = const.base_keys
    if os.getenv(const.OTLP_ENABLE_KEY, "0") == "1":
        required_keys += const.otlp_keys
    
    missing_keys = [
        key for key in required_keys 
        if os.getenv(key, None) is None or os.getenv(key, "") == ""
    ]
    
    if missing_keys:
        raise EnvNotFoundException(str(missing_keys))
```

**节来源**
- [app.py](file://core/plugin/rpa/api/app.py#L70-L95)

## 负载均衡与反向代理

### Nginx反向代理配置

系统采用Nginx作为反向代理服务器，提供负载均衡和SSL终止功能：

```mermaid
graph LR
subgraph "客户端请求"
Client[客户端]
end
subgraph "Nginx代理层"
Nginx[Nginx服务器<br/>端口: 80/443]
Static[静态资源处理]
SSE[SSE事件流]
API[API请求代理]
end
subgraph "后端服务"
RPA1[RPA服务实例1]
RPA2[RPA服务实例2]
RPA3[RPA服务实例N]
end
Client --> Nginx
Nginx --> Static
Nginx --> SSE
Nginx --> API
API --> RPA1
API --> RPA2
API --> RPA3
```

**图表来源**
- [nginx.conf](file://docker/astronAgent/nginx/nginx.conf#L1-L150)

### 反向代理配置详解

#### 静态资源处理
```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    proxy_pass http://console-frontend:1881;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

#### Server-Sent Events (SSE) 支持
```nginx
location /workflow/v1/chat/completions {
    proxy_pass http://core-workflow:7880/workflow/v1/chat/completions;
    proxy_buffering off;
    proxy_cache off;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    proxy_connect_timeout 60s;
    proxy_send_timeout 1800s;
    proxy_read_timeout 1800s;
}
```

#### API请求代理
```nginx
location /console-api/ {
    proxy_pass http://console-hub:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_connect_timeout 30s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

**节来源**
- [nginx.conf](file://docker/astronAgent/nginx/nginx.conf#L30-L120)

### SSL/TLS配置

系统支持HTTPS配置，包括Let's Encrypt证书自动获取：

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;
    
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    
    # SSL配置优化
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;
}
```

## 多实例部署配置

### Docker Compose 扩展配置

支持通过docker-compose扩展文件实现多实例部署：

```yaml
# docker-compose.override.yml
version: '3.8'

services:
  core-rpa:
    image: ghcr.io/iflytek/astron-agent/core-rpa:${ASTRON_AGENT_VERSION:-latest}
    environment:
      SERVICE_PORT: "${CORE_RPA_PORT:-17198}"
      SERVICE_WORKERS: "${SERVICE_WORKERS:-4}"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
    networks:
      - astron-agent-network
    restart: always
```

### 负载均衡策略

#### 轮询负载均衡
```nginx
upstream rpa_backend {
    least_conn;
    server core-rpa-1:17198 weight=3;
    server core-rpa-2:17198 weight=3;
    server core-rpa-3:17198 weight=2;
}
```

#### 健康检查负载均衡
```nginx
upstream rpa_backend {
    server core-rpa-1:17198 max_fails=3 fail_timeout=30s;
    server core-rpa-2:17198 max_fails=3 fail_timeout=30s;
    server core-rpa-3:17198 max_fails=3 fail_timeout=30s;
}
```

### 实例间通信

#### 服务发现机制
- **DNS服务发现**：通过容器名称解析
- **环境变量配置**：动态配置服务地址
- **健康检查集成**：自动剔除不健康实例

#### 数据一致性保证
- **共享存储**：使用MinIO存储共享文件
- **数据库连接池**：统一的数据库连接管理
- **缓存同步**：Redis集群保证缓存一致性

### 弹性伸缩配置

```yaml
deploy:
  resources:
    limits:
      cpus: '4.0'
      memory: 8G
    reservations:
      cpus: '2.0'
      memory: 4G
  restart_policy:
    condition: on-failure
    delay: 5s
    max_attempts: 3
    window: 120s
```

## 监控与可观测性

### OpenTelemetry集成

系统集成了完整的可观测性栈，支持分布式追踪、指标收集和日志聚合：

```mermaid
graph TB
subgraph "应用层"
RPA[RPA服务实例]
end
subgraph "监控代理"
OTLP[OpenTelemetry Collector]
Jaeger[Jaeger追踪]
Prometheus[Prometheus指标]
Grafana[Grafana仪表板]
end
subgraph "存储层"
JaegerStore[(Jaeger存储)]
PromStore[(Prometheus存储)]
Logs[(日志存储)]
end
RPA --> OTLP
OTLP --> Jaeger
OTLP --> Prometheus
OTLP --> Logs
Jaeger --> JaegerStore
Prometheus --> PromStore
```

**图表来源**
- [config.env](file://docker/astronAgent/config/rpa/config.env#L15-L50)

### 监控配置参数

#### 分布式追踪配置
```yaml
OTLP_ENDPOINT: ${YOUR_OTLP_ENDPOINT}
OTLP_SERVICE_NAME: RPA
OTLP_DC: hf
OTLP_ENABLE: 1
OTLP_TRACE_TIMEOUT: 3000
OTLP_TRACE_MAX_QUEUE_SIZE: 2048
OTLP_TRACE_SCHEDULE_DELAY_MILLIS: 3000
OTLP_TRACE_MAX_EXPORT_BATCH_SIZE: 500
OTLP_TRACE_EXPORT_TIMEOUT_MILLIS: 3000
```

#### 指标收集配置
```yaml
OTLP_METRIC_EXPORT_INTERVAL_MILLIS: 3000
OTLP_METRIC_EXPORT_TIMEOUT_MILLIS: 3000
OTLP_METRIC_TIMEOUT: 3000
```

**节来源**
- [config.env](file://docker/astronAgent/config/rpa/config.env#L15-L50)

### 关键监控指标

#### 应用指标
- **请求响应时间**：HTTP请求平均响应时间
- **错误率**：API调用失败百分比
- **并发连接数**：当前活跃连接数量
- **内存使用率**：应用内存占用情况

#### 基础设施指标
- **CPU使用率**：容器CPU占用情况
- **磁盘I/O**：存储读写性能
- **网络吞吐量**：网络带宽使用
- **数据库连接数**：MySQL连接池状态

#### 业务指标
- **任务执行成功率**：RPA任务完成率
- **队列积压量**：消息队列等待任务数
- **文件上传下载速率**：MinIO存储操作性能

### 告警配置

```yaml
# Prometheus告警规则
groups:
  - name: rpa_alerts
    rules:
      - alert: RPAHighErrorRate
        expr: rate(rpa_http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "RPA服务错误率过高"
          
      - alert: RPAMemoryHigh
        expr: rpa_memory_usage_percent > 80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "RPA服务内存使用率过高"
```

## 最佳实践与优化建议

### 性能优化

#### 容器资源配置
```yaml
services:
  core-rpa:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
```

#### 应用级优化
- **连接池配置**：合理设置数据库和Redis连接池大小
- **缓存策略**：使用Redis缓存热点数据
- **异步处理**：利用异步编程提高并发性能
- **资源预热**：启动时预加载常用资源

### 安全加固

#### 网络安全
```yaml
services:
  core-rpa:
    networks:
      - secure_network
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
```

#### 访问控制
- **网络隔离**：使用专用网络隔离服务
- **端口限制**：仅暴露必要端口
- **身份认证**：集成OAuth2认证机制
- **API限流**：防止API滥用攻击

### 运维最佳实践

#### 日志管理
```yaml
services:
  core-rpa:
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"
        labels: "service=rpa"
```

#### 备份策略
- **数据备份**：定期备份MySQL和Redis数据
- **配置备份**：保存Docker Compose配置文件
- **镜像备份**：定期导出和备份Docker镜像

#### 监控告警
- **实时监控**：设置关键指标的实时监控
- **自动化恢复**：配置故障自动恢复机制
- **容量规划**：基于历史数据进行容量预测

### 故障排除指南

#### 常见问题诊断
1. **服务启动失败**
   - 检查环境变量配置
   - 验证端口是否被占用
   - 查看容器日志输出

2. **连接超时问题**
   - 检查网络连通性
   - 验证防火墙设置
   - 确认服务健康状态

3. **性能瓶颈识别**
   - 监控CPU和内存使用率
   - 分析数据库查询性能
   - 检查网络延迟情况

#### 调试工具配置
```yaml
services:
  core-rpa:
    environment:
      PYTHONUNBUFFERED: 1
      DEBUG: 1
    volumes:
      - ./debug:/app/debug
    ports:
      - "5678:5678"  # PyCharm调试端口
```

通过遵循这些最佳实践和优化建议，可以确保RPA服务在生产环境中稳定、高效地运行，同时具备良好的可维护性和可扩展性。