# RPA部署配置

<cite>
**本文档引用的文件**  
- [docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)
- [docker-compose-with-auth-rpa.yaml](file://docker/astronAgent/docker-compose-with-auth-rpa.yaml)
- [astronRPA/docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml)
- [nginx/nginx.conf](file://docker/astronAgent/nginx/nginx.conf)
- [astronRPA/volumes/nginx/default.conf](file://docker/astronAgent/astronRPA/volumes/nginx/default.conf)
- [astronRPA/volumes/nginx/lua/auth_handler.lua](file://docker/astronAgent/astronRPA/volumes/nginx/lua/auth_handler.lua)
- [core/plugin/rpa/config.env](file://core/plugin/rpa/config.env)
- [config/rpa/config.env](file://docker/astronAgent/config/rpa/config.env)
</cite>

## 目录
1. [部署架构概述](#部署架构概述)
2. [Docker容器化部署](#docker容器化部署)
3. [Nginx反向代理配置](#nginx反向代理配置)
4. [环境变量配置指南](#环境变量配置指南)
5. [高可用部署建议](#高可用部署建议)
6. [性能调优技巧](#性能调优技巧)

## 部署架构概述

本RPA系统采用微服务架构，通过Docker容器化部署，实现了服务的解耦和独立扩展。系统主要由基础设施服务、核心服务和控制台服务三大部分组成。

基础设施服务包括PostgreSQL、MySQL、Redis、MinIO等，为上层应用提供数据存储和缓存支持。核心服务包括租户服务、内存数据库服务、RPA插件服务、链接插件服务、AI工具插件服务、代理服务、知识库服务和工作流服务，各服务通过REST API进行通信。控制台服务则包括Nginx反向代理、前端应用和Hub服务，为用户提供友好的操作界面。

整个系统通过Docker Compose进行编排管理，所有服务运行在同一个Docker网络（astron-agent-network）中，确保服务间的网络互通。系统支持通过docker-compose-with-auth-rpa.yaml文件集成Casdoor认证服务和astronRPA服务，实现完整的身份认证和RPA功能。

**Section sources**
- [docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)
- [docker-compose-with-auth-rpa.yaml](file://docker/astronAgent/docker-compose-with-auth-rpa.yaml)

## Docker容器化部署

### docker-compose.yml文件结构

docker-compose.yml文件定义了系统中所有服务的配置，采用YAML格式组织，主要包含服务（services）、网络（networks）和卷（volumes）三个部分。

服务部分按照功能划分为基础设施服务、核心服务和控制台服务三大类。每个服务都指定了镜像、容器名称、环境变量、依赖关系、卷挂载和网络配置。例如，RPA插件服务（core-rpa）的配置如下：

```yaml
core-rpa:
  image: ghcr.io/iflytek/astron-agent/core-rpa:${ASTRON_AGENT_VERSION:-latest}
  container_name: astron-agent-core-rpa
  environment:
    SERVICE_PORT: "${CORE_RPA_PORT:-17198}"
    OTLP_ENDPOINT: "${OTLP_ENDPOINT:-127.0.0.1:4317}"
    OTLP_ENABLE: "${OTLP_ENABLE:-0}"
    KAFKA_ENABLE: "${KAFKA_ENABLE:-0}"
    KAFKA_SERVERS: "${KAFKA_SERVERS:-kafka:29092}"
    XIAOWU_RPA_TASK_CREATE_URL: "${XIAOWU_RPA_TASK_CREATE_URL_INTERNAL:-${XIAOWU_RPA_TASK_CREATE_URL}}"
    XIAOWU_RPA_TASK_QUERY_URL: "${XIAOWU_RPA_TASK_QUERY_URL_INTERNAL:-${XIAOWU_RPA_TASK_QUERY_URL}}"
  depends_on:
    postgres:
      condition: service_healthy
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
    minio:
      condition: service_healthy
  volumes:
    - ./config/rpa/config.env:/opt/core/plugin/rpa/config.env
    - ./config/rpa/logs/:/opt/core/logs/
  networks:
    - astron-agent-network
  restart: always
```

### 服务依赖关系

服务间的依赖关系通过`depends_on`指令定义，确保服务按正确的顺序启动。所有核心服务都依赖于基础设施服务（PostgreSQL、MySQL、Redis、MinIO），并且要求这些服务处于健康状态（service_healthy）才启动。这种依赖关系确保了服务启动时所需的基础环境已经准备就绪。

### 端口映射

端口映射通过`ports`指令实现，将容器内部端口映射到主机端口。例如，Nginx服务将容器的80端口映射到主机的80端口（或通过环境变量EXPOSE_NGINX_PORT指定的端口）：

```yaml
nginx:
  ports:
    - "${EXPOSE_NGINX_PORT:-80}:80"
```

对于不需要从外部直接访问的服务，如数据库服务，通常不进行端口映射，仅在Docker网络内部通过服务名称访问。

### 卷挂载

卷挂载通过`volumes`指令实现，用于持久化数据和配置文件。系统定义了多个命名卷（named volumes），如postgres_data、mysql_data、redis_data等，用于持久化数据库数据。同时，将本地配置文件和日志目录挂载到容器中，实现配置的外部化和日志的持久化存储。

例如，RPA服务的配置文件和日志目录挂载如下：

```yaml
volumes:
  - ./config/rpa/config.env:/opt/core/plugin/rpa/config.env
  - ./config/rpa/logs/:/opt/core/logs/
```

这种配置方式使得配置修改无需重建镜像，日志文件也能够持久保存在主机上，便于问题排查。

**Section sources**
- [docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)
- [astronRPA/docker-compose.yml](file://docker/astronAgent/astronRPA/docker-compose.yml)

## Nginx反向代理配置

### Nginx配置文件结构

Nginx配置文件（nginx.conf）采用模块化结构，主要包含events和http两个顶级块。events块配置了工作进程的连接数，http块包含了MIME类型定义、日志格式、基本配置、Gzip压缩和服务器配置。

服务器配置（server块）监听80端口，定义了多个location块来处理不同路径的请求。配置文件通过include指令包含了MIME类型定义，并设置了访问日志和错误日志。

### Lua脚本认证机制

在astronRPA的Nginx配置中，使用了OpenResty和Lua脚本来实现请求拦截和验证。通过`access_by_lua_file`指令，将特定路径的请求交给Lua脚本处理：

```nginx
location /api/resource/ {
    access_by_lua_file lua/auth_handler.lua;
    proxy_pass http://resource-service;
    # ... 其他代理配置
}
```

### auth_handler.lua认证流程

auth_handler.lua脚本实现了完整的认证流程，主要步骤如下：

1. **令牌提取**：从请求头中提取Authorization或自定义的Token头作为会话令牌。
2. **令牌验证**：通过HTTP请求将令牌发送给robot-service服务的/user/info接口进行验证。
3. **响应处理**：解析robot-service的响应，检查认证结果。
4. **请求继续或拒绝**：根据认证结果决定是否继续处理请求或返回错误。

认证成功后，脚本会设置user_id和user-info请求头，供后端服务使用：

```lua
ngx.req.set_header("user_id", user_id)
ngx.req.set_header("user-info", json.encode({id = user_id}))
```

这种基于Lua脚本的认证机制具有高度的灵活性，可以根据业务需求实现复杂的认证逻辑，同时保持Nginx的高性能。

**Section sources**
- [nginx/nginx.conf](file://docker/astronAgent/nginx/nginx.conf)
- [astronRPA/volumes/nginx/default.conf](file://docker/astronAgent/astronRPA/volumes/nginx/default.conf)
- [astronRPA/volumes/nginx/lua/auth_handler.lua](file://docker/astronAgent/astronRPA/volumes/nginx/lua/auth_handler.lua)

## 环境变量配置指南

### RPA服务环境变量

RPA服务的环境变量主要在config.env文件中定义，分为服务信息、日志、可观测性、消息队列和外部服务配置等类别。

#### 服务信息配置

- **SERVICE_PORT**: 服务监听端口，默认17198
- **SERVICE_LOCATION**: 服务位置标识，用于区分不同数据中心的部署
- **SERVICE_NAME**: 服务名称，用于服务发现和监控

#### 可观测性配置

- **OTLP_ENABLE**: 是否启用OpenTelemetry，1为启用，0为禁用
- **OTLP_ENDPOINT**: OTLP服务端点，用于上报追踪和指标数据
- **OTLP_METRIC_EXPORT_INTERVAL_MILLIS**: 指标上报间隔，推荐小于30000ms
- **OTLP_TRACE_MAX_EXPORT_BATCH_SIZE**: 追踪数据最大批量大小

#### 消息队列配置

- **KAFKA_ENABLE**: 是否启用Kafka，1为启用，0为禁用
- **KAFKA_SERVERS**: Kafka服务器地址，多个地址用逗号分隔
- **KAFKA_TOPIC**: Kafka主题名称

#### 外部服务配置

- **XIAOWU_RPA_TIMEOUT**: XiaoWu RPA超时时间，单位毫秒
- **XIAOWU_RPA_PING_INTERVAL**: XiaoWu RPA心跳间隔，单位秒
- **XIAOWU_RPA_TASK_QUERY_INTERVAL**: RPA任务查询间隔，单位秒
- **XIAOWU_RPA_TASK_CREATE_URL**: RPA任务创建URL
- **XIAOWU_RPA_TASK_QUERY_URL**: RPA任务查询URL

### 关键参数推荐值

| 参数名称 | 推荐值 | 说明 |
|--------|-------|------|
| XIAOWU_RPA_TIMEOUT_KEY | 3000 | RPA操作超时时间，单位毫秒 |
| XIAOWU_RPA_TASK_QUERY_INTERVAL_KEY | 10 | 任务状态查询间隔，单位秒 |
| OTLP_METRIC_EXPORT_INTERVAL_MILLIS | 3000 | 指标上报间隔，单位毫秒 |
| OTLP_TRACE_SCHEDULE_DELAY_MILLIS | 3000 | 追踪数据上报延迟，单位毫秒 |
| KAFKA_TIMEOUT | 10 | Kafka操作超时，单位秒 |

这些参数的合理配置对于系统的稳定性和性能至关重要。建议根据实际部署环境和业务需求进行调整，并通过监控系统持续观察其效果。

**Section sources**
- [core/plugin/rpa/config.env](file://core/plugin/rpa/config.env)
- [config/rpa/config.env](file://docker/astronAgent/config/rpa/config.env)

## 高可用部署建议

为确保RPA系统的高可用性，建议采取以下部署策略：

1. **多节点部署**：将关键服务（如数据库、Redis、MinIO）部署在多个节点上，避免单点故障。可以使用Docker Swarm或Kubernetes等容器编排工具实现服务的多副本部署和自动故障转移。

2. **负载均衡**：在Nginx前端部署负载均衡器，将流量分发到多个实例。可以使用硬件负载均衡器或云服务商提供的负载均衡服务。

3. **数据持久化和备份**：确保所有数据卷都正确挂载到持久化存储，并定期备份关键数据。对于数据库，建议配置主从复制和定期备份策略。

4. **健康检查**：充分利用Docker Compose的healthcheck功能，为每个服务配置合理的健康检查。这可以确保只有健康的实例才会接收流量。

5. **监控和告警**：部署全面的监控系统，监控CPU、内存、磁盘、网络等系统指标，以及服务的响应时间、错误率等业务指标。配置合理的告警阈值，及时发现和处理问题。

6. **灾难恢复**：制定详细的灾难恢复计划，包括数据恢复、服务重建和流量切换等步骤。定期进行灾难恢复演练，确保计划的有效性。

通过实施这些高可用策略，可以显著提高系统的稳定性和可靠性，确保业务连续性。

**Section sources**
- [docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)

## 性能调优技巧

为优化RPA系统的性能，建议采取以下调优措施：

1. **JVM参数优化**：对于Java服务（如console-hub），合理配置JVM参数，如堆内存大小、垃圾回收器等。可以参考Dockerfile中的JVM参数配置：

```dockerfile
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:+UseG1GC", \
    "-XX:+UseStringDeduplication", \
    "-jar", "/app/app.jar"]
```

2. **数据库优化**：为数据库表创建适当的索引，优化查询语句。定期分析和优化数据库性能，如调整缓冲区大小、连接池大小等。

3. **缓存策略**：充分利用Redis缓存，减少数据库访问。对于频繁读取但不经常变化的数据，可以设置较长的缓存时间。

4. **连接池配置**：合理配置数据库和HTTP客户端的连接池大小，避免连接过多导致资源耗尽，或连接过少导致性能瓶颈。

5. **异步处理**：对于耗时较长的操作，如RPA任务执行，采用异步处理模式，避免阻塞主线程。可以使用Kafka等消息队列实现异步通信。

6. **资源限制**：为每个容器设置合理的资源限制（CPU和内存），避免单个服务占用过多资源影响其他服务。

7. **日志级别**：在生产环境中，将日志级别设置为INFO或WARN，减少日志输出对性能的影响。对于调试信息，可以使用条件日志或动态调整日志级别。

通过综合运用这些性能调优技巧，可以显著提升系统的响应速度和吞吐量，提供更好的用户体验。

**Section sources**
- [console/backend/hub/Dockerfile](file://console/backend/hub/Dockerfile)
- [docker-compose.yaml](file://docker/astronAgent/docker-compose.yaml)