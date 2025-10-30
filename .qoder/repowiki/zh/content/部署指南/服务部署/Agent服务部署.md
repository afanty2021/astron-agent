# Agent服务部署文档

<cite>
**本文档中引用的文件**
- [core/agent/Dockerfile](file://core/agent/Dockerfile)
- [console/frontend/Dockerfile](file://console/frontend/Dockerfile)
- [docker/astronAgent/docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)
- [core/agent/main.py](file://core/agent/main.py)
- [console/frontend/docker-entrypoint.sh](file://console/frontend/docker-entrypoint.sh)
- [core/agent/config.env.example](file://core/agent/config.env.example)
- [console/frontend/nginx.conf](file://console/frontend/nginx.conf)
- [core/agent/api/app.py](file://core/agent/api/app.py)
- [core/workflow/main.py](file://core/workflow/main.py)
- [core/knowledge/main.py](file://core/knowledge/main.py)
- [core/agent/pyproject.toml](file://core/agent/pyproject.toml)
</cite>

## 目录
1. [概述](#概述)
2. [项目架构](#项目架构)
3. [Dockerfile详解](#dockerfile详解)
4. [环境配置](#环境配置)
5. [服务依赖关系](#服务依赖关系)
6. [部署流程](#部署流程)
7. [健康检查配置](#健康检查配置)
8. [日志管理](#日志管理)
9. [性能调优](#性能调优)
10. [故障排除](#故障排除)
11. [最佳实践](#最佳实践)

## 概述

astron-agent是一个基于Python的智能代理服务系统，采用微服务架构设计，支持多种插件和工具集成。Agent服务作为核心组件，负责处理自然语言理解、工作流执行和知识检索等核心功能。

### 核心特性
- **模块化设计**：支持插件化扩展
- **多语言支持**：基于FastAPI框架
- **分布式架构**：支持集群部署
- **可观测性**：内置监控和追踪
- **高可用性**：支持健康检查和服务发现

## 项目架构

```mermaid
graph TB
subgraph "客户端层"
WebUI[Web界面]
API[API客户端]
end
subgraph "网关层"
Nginx[Nginx反向代理]
end
subgraph "应用服务层"
ConsoleFrontend[控制台前端]
ConsoleHub[控制台后端]
AgentService[Agent服务]
WorkflowService[工作流服务]
KnowledgeService[知识库服务]
TenantService[租户服务]
DatabaseService[数据库服务]
RPAServer[RPA服务]
LinkServer[链接服务]
AIToolsServer[AI工具服务]
end
subgraph "基础设施层"
MySQL[(MySQL数据库)]
PostgreSQL[(PostgreSQL数据库)]
Redis[(Redis缓存)]
MinIO[(对象存储)]
Kafka[(消息队列)]
end
WebUI --> Nginx
API --> Nginx
Nginx --> ConsoleFrontend
Nginx --> ConsoleHub
ConsoleHub --> AgentService
ConsoleHub --> WorkflowService
ConsoleHub --> KnowledgeService
ConsoleHub --> TenantService
ConsoleHub --> DatabaseService
ConsoleHub --> RPAServer
ConsoleHub --> LinkServer
ConsoleHub --> AIToolsServer
AgentService --> MySQL
AgentService --> PostgreSQL
AgentService --> Redis
AgentService --> MinIO
AgentService --> Kafka
WorkflowService --> MySQL
WorkflowService --> PostgreSQL
WorkflowService --> Redis
WorkflowService --> MinIO
WorkflowService --> Kafka
KnowledgeService --> MySQL
KnowledgeService --> PostgreSQL
KnowledgeService --> Redis
KnowledgeService --> MinIO
KnowledgeService --> Kafka
```

**图表来源**
- [docker/astronAgent/docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml#L1-L614)

## Dockerfile详解

### Agent服务Dockerfile分析

Agent服务使用Python 3.11 Slim作为基础镜像，采用多阶段构建策略优化镜像大小。

#### 基础配置
- **基础镜像**：`python:3.11-slim`
- **工作目录**：`/opt/core/agent`
- **环境变量**：
  - `PATH`：添加到系统路径
  - `PYTHONPATH`：设置Python模块搜索路径

#### 依赖管理
- **包管理器**：使用uv替代pip，提高安装效率
- **依赖安装**：
  - 先安装通用依赖（common）
  - 再安装Agent特定依赖
  - 使用清华镜像加速下载

#### 构建流程
```mermaid
flowchart TD
Start([开始构建]) --> BaseImage["使用Python 3.11 Slim"]
BaseImage --> SetEnv["设置环境变量"]
SetEnv --> InstallUV["安装uv包管理器"]
InstallUV --> CopyCommon["复制common模块"]
CopyCommon --> CopyAgent["复制Agent代码"]
CopyAgent --> CopyConfig["复制配置文件"]
CopyConfig --> InstallDeps["安装依赖"]
InstallDeps --> CMD["设置启动命令"]
CMD --> End([构建完成])
```

**图表来源**
- [core/agent/Dockerfile](file://core/agent/Dockerfile#L1-L24)

**节来源**
- [core/agent/Dockerfile](file://core/agent/Dockerfile#L1-L24)

### 前端服务Dockerfile分析

前端服务采用多阶段构建，先构建静态资源，再部署到Nginx。

#### 构建阶段
- **构建器镜像**：`node:18-alpine`
- **工作目录**：`/app`
- **依赖缓存**：使用BuildKit缓存机制

#### 运行时配置
- **运行镜像**：`nginx:1.15-alpine`
- **端口配置**：默认80端口
- **性能优化**：启用gzip压缩，配置worker进程数

**节来源**
- [console/frontend/Dockerfile](file://console/frontend/Dockerfile#L1-L81)

## 环境配置

### Agent服务配置文件

Agent服务通过`config.env`文件管理所有环境变量，支持开发、测试和生产环境。

#### 核心配置项

| 配置项 | 描述 | 默认值 | 必需 |
|--------|------|--------|------|
| `SERVICE_HOST` | 服务绑定地址 | `0.0.0.0` | 是 |
| `SERVICE_PORT` | 服务端口号 | `17870` | 是 |
| `SERVICE_WORKERS` | 工作进程数 | `1` | 否 |
| `REDIS_ADDR` | Redis服务器地址 | - | 是 |
| `MYSQL_HOST` | MySQL主机地址 | - | 是 |
| `OTLP_ENABLE` | 启用遥测报告 | `1` | 否 |

#### 数据库配置
- **MySQL配置**：支持主从复制
- **Redis配置**：支持集群模式
- **连接池**：自动管理连接复用

#### 监控配置
- **OTLP端点**：支持分布式追踪
- **指标收集**：实时性能监控
- **链路追踪**：请求链路可视化

**节来源**
- [core/agent/config.env.example](file://core/agent/config.env.example#L1-L97)

### 环境变量管理

```mermaid
sequenceDiagram
participant Container as 容器启动
participant Main as main.py
participant Env as 环境加载器
participant Config as 配置文件
Container->>Main : 启动服务
Main->>Env : 加载环境变量
Env->>Config : 读取config.env
Config-->>Env : 返回配置值
Env->>Env : 验证配置完整性
Env-->>Main : 返回环境变量
Main->>Main : 初始化服务
Main-->>Container : 服务就绪
```

**图表来源**
- [core/agent/main.py](file://core/agent/main.py#L1-L110)

**节来源**
- [core/agent/main.py](file://core/agent/main.py#L1-L110)

## 服务依赖关系

### Agent服务依赖图

```mermaid
graph LR
subgraph "Agent服务"
Agent[Agent服务]
end
subgraph "外部服务"
Link[Link服务<br/>18888端口]
Workflow[Workflow服务<br/>7880端口]
Knowledge[Knowledge服务<br/>20010端口]
Tenant[Tenant服务<br/>5052端口]
end
subgraph "基础设施"
MySQL[(MySQL<br/>3306端口)]
Redis[(Redis<br/>6379端口)]
MinIO[(MinIO<br/>9000端口)]
end
Agent --> Link
Agent --> Workflow
Agent --> Knowledge
Agent --> Tenant
Agent --> MySQL
Agent --> Redis
Agent --> MinIO
```

**图表来源**
- [docker/astronAgent/docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml#L200-L350)

### 通信协议

Agent服务通过HTTP/HTTPS与下游服务通信，支持以下协议：

1. **RESTful API**：标准HTTP请求
2. **WebSocket**：实时通信
3. **gRPC**：高性能RPC调用
4. **消息队列**：异步事件处理

**节来源**
- [docker/astronAgent/docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml#L200-L350)

## 部署流程

### 本地开发部署

#### 1. 环境准备
```bash
# 克隆仓库
git clone https://github.com/iflytek/astron-agent.git
cd astron-agent

# 创建虚拟环境
python -m venv venv
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt
```

#### 2. 配置环境
```bash
# 复制配置文件
cp core/agent/config.env.example core/agent/config.env

# 编辑配置文件
nano core/agent/config.env
```

#### 3. 启动服务
```bash
# 启动Agent服务
cd core/agent
python main.py
```

### Docker部署

#### 1. 构建镜像
```bash
# 构建Agent服务镜像
cd core/agent
docker build -t astron-agent:latest .

# 构建前端服务镜像
cd console/frontend
docker build -t astron-agent-frontend:latest .
```

#### 2. 使用Docker Compose
```bash
# 启动完整服务栈
cd docker/astronAgent
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f
```

### 生产环境部署

#### 1. 集群部署配置
```yaml
# docker-compose.prod.yaml
version: '3.8'
services:
  agent:
    image: ghcr.io/iflytek/astron-agent/core-agent:${VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
```

#### 2. 负载均衡配置
```nginx
upstream astron_agent {
    server agent1:17870;
    server agent2:17870;
    server agent3:17870;
}

server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://astron_agent;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**节来源**
- [docker/astronAgent/docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml#L1-L614)

## 健康检查配置

### Agent服务健康检查

Agent服务实现了多层次的健康检查机制：

#### 1. 应用级健康检查
```python
# 健康检查端点
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "services": {
            "database": check_database_health(),
            "redis": check_redis_health(),
            "external": check_external_services_health()
        }
    }
```

#### 2. 服务依赖检查
- **数据库连接**：检查MySQL和PostgreSQL连接
- **缓存连接**：验证Redis连接状态
- **外部服务**：确认下游服务可达性

#### 3. Docker健康检查配置
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:17870/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**节来源**
- [core/agent/api/app.py](file://core/agent/api/app.py#L1-L85)

### 监控指标

Agent服务提供以下监控指标：

| 指标类别 | 指标名称 | 描述 |
|----------|----------|------|
| 性能指标 | 请求响应时间 | 平均响应时间分布 |
| 性能指标 | 并发请求数 | 当前活跃连接数 |
| 错误指标 | HTTP错误率 | 4xx/5xx错误统计 |
| 资源指标 | CPU使用率 | 系统CPU占用情况 |
| 资源指标 | 内存使用率 | 内存占用统计 |
| 业务指标 | 代理处理量 | 成功处理的请求数量 |

## 日志管理

### 日志配置结构

```mermaid
graph TB
subgraph "日志层级"
App[应用日志]
Access[访问日志]
Error[错误日志]
Debug[调试日志]
end
subgraph "日志格式"
Structured[结构化日志]
JSON[JSON格式]
Plain[纯文本]
end
subgraph "日志存储"
Local[本地文件]
Remote[远程存储]
Centralized[集中化]
end
App --> Structured
Access --> JSON
Error --> Plain
Debug --> JSON
Structured --> Local
JSON --> Remote
Plain --> Centralized
```

### 日志级别配置

Agent服务支持以下日志级别：

| 级别 | 用途 | 示例场景 |
|------|------|----------|
| DEBUG | 详细调试信息 | 开发阶段问题排查 |
| INFO | 一般信息记录 | 正常操作日志 |
| WARNING | 警告信息 | 配置不当或性能警告 |
| ERROR | 错误信息 | 运行时异常和错误 |
| CRITICAL | 严重错误 | 系统级故障 |

### 日志轮转配置

```nginx
# Nginx日志配置
access_log /var/log/nginx/access.log main;
error_log /var/log/nginx/error.log warn;

# 日志轮转
logrotate /etc/logrotate.d/nginx {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 644 nginx nginx
}
```

**节来源**
- [console/frontend/nginx.conf](file://console/frontend/nginx.conf#L1-L23)

## 性能调优

### 系统级优化

#### 1. 内存优化
```yaml
# Docker内存限制
services:
  agent:
    mem_limit: 4g
    memswap_limit: 8g
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
```

#### 2. CPU优化
```python
# 工作进程配置
import multiprocessing
workers = multiprocessing.cpu_count() * 2
```

#### 3. 网络优化
```nginx
# Nginx网络配置
worker_connections 1024;
keepalive_timeout 65;
client_max_body_size 100M;
```

### 应用级优化

#### 1. 连接池配置
```python
# SQLAlchemy连接池
from sqlalchemy import create_engine
engine = create_engine(
    "mysql+pymysql://user:pass@host/db",
    pool_size=20,
    max_overflow=30,
    pool_pre_ping=True
)
```

#### 2. 缓存策略
```python
# Redis缓存配置
redis = Redis.from_url("redis://localhost:6379/0")
redis.client_setname("agent-service")
redis.expire("cache_key", 3600)  # 1小时过期
```

#### 3. 异步处理
```python
# 异步任务队列
from celery import Celery
celery = Celery('tasks', broker='redis://localhost:6379/0')

@celery.task
def process_large_task(data):
    # 大数据处理逻辑
    pass
```

### 性能监控指标

| 指标类型 | 监控项 | 告警阈值 | 优化建议 |
|----------|--------|----------|----------|
| 响应时间 | 平均响应时间 | >500ms | 优化数据库查询 |
| 响应时间 | P95响应时间 | >1000ms | 扩容或优化 |
| 错误率 | HTTP 5xx错误 | >1% | 检查服务稳定性 |
| 资源使用 | CPU使用率 | >80% | 增加实例数量 |
| 资源使用 | 内存使用率 | >85% | 优化内存使用 |
| 并发数 | 活跃连接数 | >80%最大值 | 调整连接池 |

## 故障排除

### 常见问题及解决方案

#### 1. 服务启动失败

**问题现象**：Agent服务无法启动
**可能原因**：
- 端口被占用
- 配置文件错误
- 依赖服务不可用

**解决步骤**：
```bash
# 检查端口占用
netstat -tulpn | grep 17870

# 检查配置文件语法
python -m json.tool config.env

# 检查依赖服务状态
docker-compose ps
```

#### 2. 数据库连接问题

**问题现象**：数据库连接超时
**诊断命令**：
```bash
# 检查数据库服务状态
docker-compose exec mysql mysql -uroot -p

# 测试连接
telnet mysql 3306
```

#### 3. 内存不足

**问题现象**：服务频繁重启
**解决方案**：
```yaml
# 增加内存限制
services:
  agent:
    mem_limit: 8g
    memswap_limit: 16g
```

#### 4. 性能瓶颈

**问题现象**：响应时间过长
**诊断方法**：
```bash
# 监控CPU和内存
docker stats

# 分析慢查询
tail -f /var/log/mysql/slow.log
```

### 故障恢复流程

```mermaid
flowchart TD
Alert[收到告警] --> Assess[评估影响范围]
Assess --> Isolate[隔离故障服务]
Isolate --> Diagnose[诊断根本原因]
Diagnose --> Fix[实施修复方案]
Fix --> Test[测试修复效果]
Test --> Monitor[持续监控]
Monitor --> Document[记录故障过程]
Document --> Improve[改进预防措施]
```

## 最佳实践

### 部署最佳实践

#### 1. 环境分离
- 开发环境：使用本地数据库和缓存
- 测试环境：使用独立的测试数据库
- 生产环境：使用高可用基础设施

#### 2. 安全配置
```yaml
# 安全配置示例
environment:
  # 禁用调试模式
  FLASK_ENV: production
  
  # 设置安全头
  SECURE_HEADERS: true
  
  # 限制文件上传大小
  MAX_CONTENT_LENGTH: 100MB
```

#### 3. 监控告警
```yaml
# Prometheus监控配置
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
```

### 运维最佳实践

#### 1. 版本管理
- 使用语义化版本号
- 维护变更日志
- 实施蓝绿部署

#### 2. 备份策略
```bash
# 自动备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
docker exec mysql mysqldump -uuser -ppass database > backup_$DATE.sql
tar -czf backup_$DATE.tar.gz backup_$DATE.sql
```

#### 3. 容量规划
- 监控资源使用趋势
- 预留足够的缓冲空间
- 实施自动扩缩容

### 开发最佳实践

#### 1. 代码质量
- 遵循PEP 8编码规范
- 实施单元测试覆盖率要求
- 使用静态代码分析工具

#### 2. 文档维护
- 保持API文档同步更新
- 记录重要的配置变更
- 维护故障排除指南

#### 3. 团队协作
- 使用Git分支策略
- 实施代码审查流程
- 建立知识共享机制

通过遵循这些最佳实践，可以确保Agent服务的稳定运行和高效维护，为用户提供可靠的智能代理服务体验。