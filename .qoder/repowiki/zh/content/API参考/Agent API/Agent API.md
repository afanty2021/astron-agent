# Agent API

<cite>
**本文档引用的文件**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py)
- [bot_config_mgr_api.py](file://core/agent/api/v1/bot_config_mgr_api.py)
- [openapi.py](file://core/agent/api/v1/openapi.py)
- [app.py](file://core/agent/api/app.py)
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py)
- [bot_config.py](file://core/agent/api/schemas/bot_config.py)
- [base_inputs.py](file://core/agent/api/schemas/base_inputs.py)
- [completion.py](file://core/agent/api/schemas/completion.py)
- [completion_chunk.py](file://core/agent/api/schemas/completion_chunk.py)
- [agent_response.py](file://core/agent/api/schemas/agent_response.py)
- [codes.py](file://core/agent/exceptions/codes.py)
- [middleware.py](file://core/agent/infra/config/middleware.py)
- [workflow_agent_builder.py](file://core/agent/service/builder/workflow_agent_builder.py)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概述](#架构概述)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介
Agent API 文档详细介绍了智能体服务的API接口，重点涵盖工作流执行、Bot配置管理和OpenAPI接口。API基于FastAPI框架构建，提供流式和非流式响应模式，支持复杂的智能体工作流执行。系统通过分布式追踪、指标收集和错误处理机制确保高可用性和可观测性。

## 项目结构
Agent服务位于core/agent目录下，采用分层架构设计。API层位于api目录，包含v1版本的端点定义；schemas目录定义了所有请求和响应的数据模型；service目录包含业务逻辑构建器；engine目录实现核心执行引擎；exceptions目录管理错误码和异常处理。

```mermaid
graph TB
subgraph "API层"
workflow_agent[workflow_agent.py]
bot_config_mgr[bot_config_mgr_api.py]
openapi[openapi.py]
app[app.py]
end
subgraph "数据模型"
schemas[api/schemas]
base_inputs[base_inputs.py]
workflow_inputs[workflow_agent_inputs.py]
bot_config[bot_config.py]
completion[completion.py]
completion_chunk[completion_chunk.py]
end
subgraph "服务层"
service[service]
builder[builder]
workflow_builder[workflow_agent_builder.py]
openapi_builder[openapi_builder.py]
end
subgraph "引擎层"
engine[engine]
workflow_runner[workflow_agent_runner.py]
end
subgraph "异常处理"
exceptions[exceptions]
codes[codes.py]
agent_exc[agent_exc.py]
end
workflow_agent --> schemas
bot_config_mgr --> schemas
openapi --> schemas
app --> workflow_agent
app --> bot_config_mgr
app --> openapi
workflow_agent --> service
service --> engine
service --> exceptions
```

**图源**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py)
- [bot_config_mgr_api.py](file://core/agent/api/v1/bot_config_mgr_api.py)
- [openapi.py](file://core/agent/api/v1/openapi.py)
- [app.py](file://core/agent/api/app.py)

**章节源**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L105)
- [bot_config_mgr_api.py](file://core/agent/api/v1/bot_config_mgr_api.py#L1-L211)
- [openapi.py](file://core/agent/api/v1/openapi.py#L1-L209)

## 核心组件
Agent API的核心组件包括工作流执行引擎、Bot配置管理器和OpenAPI兼容接口。工作流执行引擎支持复杂的思维链(CoT)推理过程，集成知识库检索、工具调用和工作流执行功能。Bot配置管理器提供对智能体配置的CRUD操作，支持配置的持久化存储。OpenAPI接口确保与标准OpenAI API的兼容性，便于现有应用集成。

**章节源**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L105)
- [bot_config_mgr_api.py](file://core/agent/api/v1/bot_config_mgr_api.py#L1-L211)
- [openapi.py](file://core/agent/api/v1/openapi.py#L1-L209)

## 架构概述
Agent服务采用微服务架构，通过FastAPI提供RESTful API接口。系统集成分布式追踪(OTLP)、指标收集和日志记录功能，确保系统的可观测性。API层与业务逻辑层分离，通过构建器模式创建执行器实例。配置管理基于Polaris系统，支持动态配置更新。

```mermaid
graph TB
Client[客户端] --> API[Agent API]
API --> Auth[认证服务]
API --> Tracing[分布式追踪]
API --> Metrics[指标收集]
API --> Logging[日志服务]
subgraph "API层"
API --> Workflow[工作流执行]
API --> BotConfig[Bot配置管理]
API --> OpenAPI[OpenAPI兼容]
end
subgraph "业务逻辑"
Workflow --> Builder[构建器]
BotConfig --> Repository[配置仓库]
OpenAPI --> Runner[执行器]
end
subgraph "数据存储"
Repository --> MySQL[MySQL]
Repository --> Redis[Redis]
end
Builder --> LLM[大语言模型]
Builder --> Knowledge[知识库]
Builder --> Tools[工具服务]
Builder --> Workflows[工作流服务]
style Client fill:#f9f,stroke:#333
style API fill:#bbf,stroke:#333
style LLM fill:#f96,stroke:#333
```

**图源**
- [app.py](file://core/agent/api/app.py#L1-L84)
- [middleware.py](file://core/agent/infra/config/middleware.py#L1-L63)

## 详细组件分析

### 工作流执行API分析
工作流执行API提供智能体的核心执行功能，支持自定义模型配置、插件集成和思维链推理。API通过流式响应提供实时结果传输，适用于对话式应用场景。

#### 工作流执行类图
```mermaid
classDiagram
class CustomChatCompletion {
+app_id : str
+bot_id : str
+uid : str
+question : str
+span : Span
+__init__(inputs : CustomCompletionInputs, **data : Any)
+build_runner(span : Span) WorkflowAgentRunner
+do_complete() AsyncGenerator[str, None]
}
class CompletionBase {
+app_id : str
+inputs : BaseInputs
+log_caller : str
+build_runner(span : Span) Any
+build_node_trace(bot_id : str, span : Span) NodeTracePatch
+build_meter(span : Span) Meter
+_process_chunk(chunk : Any, chunk_logs : List[str]) AsyncGenerator[str, None]
+_finalize_run(context : RunContext) AsyncGenerator[str, None]
+run_runner(node_trace : NodeTrace, meter : Meter, span : Span) AsyncGenerator[str, None]
+create_chunk(chunk : Any) str
+create_stop(span : Span, e : BaseExc) ReasonChatCompletionChunk
+create_done() str
}
class CustomCompletionInputs {
+model_config_inputs : CustomCompletionModelConfigInputs
+instruction : CustomCompletionInstructionInputs
+plugin : CustomCompletionPluginInputs
+max_loop_count : int
+get_last_message_content() str
+get_chat_history() list[LLMMessage]
}
CustomChatCompletion --> CompletionBase : "继承"
CustomChatCompletion --> CustomCompletionInputs : "使用"
CompletionBase --> BaseInputs : "继承"
```

**图源**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L105)
- [base_api.py](file://core/agent/api/v1/base_api.py#L1-L226)
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py#L1-L50)

#### 工作流执行序列图
```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "WorkflowAgentAPI"
participant Builder as "WorkflowAgentBuilder"
participant Runner as "WorkflowAgentRunner"
participant LLM as "大语言模型"
participant Knowledge as "知识库"
participant Tools as "工具服务"
Client->>API : POST /agent/v1/custom/chat/completions
API->>API : 创建CustomChatCompletion实例
API->>Builder : build_runner()
Builder->>Builder : 创建模型实例
Builder->>Builder : 构建插件列表
Builder->>Builder : 查询知识库
Builder->>Builder : 创建执行器参数
Builder-->>API : 返回WorkflowAgentRunner
API->>Runner : run_runner()
Runner->>LLM : 执行思维链推理
alt 知识库查询
Runner->>Knowledge : 查询相关知识
Knowledge-->>Runner : 返回知识片段
end
alt 工具调用
Runner->>Tools : 调用指定工具
Tools-->>Runner : 返回工具结果
end
Runner->>LLM : 生成最终响应
loop 流式响应
Runner-->>API : 发送响应片段
API-->>Client : 流式传输
end
Runner-->>API : 发送[DONE]
API-->>Client : 完成响应
```

**图源**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L105)
- [workflow_agent_builder.py](file://core/agent/service/builder/workflow_agent_builder.py#L1-L230)

### Bot配置管理API分析
Bot配置管理API提供对智能体配置的完整生命周期管理，包括创建、读取、更新和删除操作。API集成分布式追踪，确保每个操作的可追溯性。

#### Bot配置管理序列图
```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "BotConfigMgrAPI"
participant Handler as "操作处理器"
participant Tracing as "追踪上下文"
participant Repository as "配置仓库"
Client->>API : POST /agent/v1/bot-config
API->>Handler : handle_bot_config_operation()
Handler->>Tracing : tracing_context()
Tracing->>Tracing : 创建Span
Tracing->>Repository : 执行创建操作
Repository-->>Tracing : 返回结果
Tracing->>Tracing : 记录输入输出
Tracing-->>Handler : 返回响应
Handler-->>API : GeneralResponse
API-->>Client : 返回创建结果
Note over Tracing,Repository : 统一的追踪上下文管理
```

**图源**
- [bot_config_mgr_api.py](file://core/agent/api/v1/bot_config_mgr_api.py#L1-L211)
- [bot_config.py](file://core/agent/api/schemas/bot_config.py#L1-L58)

### OpenAPI接口分析
OpenAPI接口提供与标准OpenAI API兼容的端点，支持流式和非流式响应模式。接口设计确保与现有OpenAI客户端的无缝集成。

#### OpenAPI接口序列图
```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "OpenAPI"
participant Builder as "OpenAPIBuilder"
participant Runner as "OpenAPIRunner"
participant LLM as "大语言模型"
Client->>API : POST /agent/v1/chat/completions
API->>API : 创建ChatCompletion实例
API->>Builder : build_runner()
Builder-->>API : 返回OpenAPIRunner
API->>Runner : run_runner()
alt 流式响应
Runner-->>API : 发送流式片段
API-->>Client : text/event-stream
else 非流式响应
Runner-->>API : 收集完整响应
API-->>Client : JSON响应
end
```

**图源**
- [openapi.py](file://core/agent/api/v1/openapi.py#L1-L209)
- [openapi_builder.py](file://core/agent/service/builder/openapi_builder.py#L1-L100)

## 依赖分析
Agent服务依赖多个内部和外部组件，形成复杂的依赖网络。核心依赖包括分布式追踪系统、配置管理服务、数据库存储和第三方API服务。

```mermaid
graph TD
workflow_agent --> base_api
workflow_agent --> workflow_agent_inputs
workflow_agent --> workflow_agent_builder
workflow_agent --> workflow_agent_runner
bot_config_mgr --> bot_config
bot_config_mgr --> bot_config_client
bot_config_mgr --> tracing_context
openapi --> base_api
openapi --> openapi_inputs
openapi --> openapi_builder
openapi --> openapi_runner
base_api --> completion_chunk
base_api --> node_trace_patch
base_api --> common_imports
workflow_agent_builder --> knowledge_plugin
workflow_agent_builder --> model_factory
workflow_agent_builder --> plugin_factory
style workflow_agent fill:#f96,stroke:#333
style bot_config_mgr fill:#6f9,stroke:#333
style openapi fill:#96f,stroke:#333
```

**图源**
- [app.py](file://core/agent/api/app.py#L1-L84)
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L105)
- [bot_config_mgr_api.py](file://core/agent/api/v1/bot_config_mgr_api.py#L1-L211)
- [openapi.py](file://core/agent/api/v1/openapi.py#L1-L209)

**章节源**
- [app.py](file://core/agent/api/app.py#L1-L84)
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L105)

## 性能考虑
Agent服务在设计时考虑了多项性能优化措施。流式响应减少客户端等待时间，分布式追踪和指标收集帮助识别性能瓶颈。缓存机制减少重复计算，异步I/O提高并发处理能力。系统支持水平扩展，通过增加工作进程处理高并发请求。

## 故障排除指南
当遇到API调用问题时，首先检查错误码和消息。常见的错误包括认证失败(40040)、配置无效(40003)和模型调用失败(40029)。启用详细的追踪日志可以帮助诊断问题根源。对于流式响应中断，检查网络连接和超时设置。

**章节源**
- [codes.py](file://core/agent/exceptions/codes.py#L1-L177)
- [base_api.py](file://core/agent/api/v1/base_api.py#L1-L226)

## 结论
Agent API提供了一套完整的智能体管理和服务执行接口，支持复杂的工作流执行、灵活的配置管理和标准的OpenAPI兼容性。系统设计注重可观测性、可扩展性和可靠性，适用于各种智能体应用场景。通过合理的错误处理和性能优化，确保服务的高可用性和响应性。