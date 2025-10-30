# 调试API

<cite>
**本文档中引用的文件**  
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py)
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py)
- [workflow_agent_runner.py](file://core/agent/engine/workflow_agent_runner.py)
- [debug.py](file://core/workflow/api/v1/chat/debug.py)
- [node_debug.py](file://core/workflow/api/v1/chat/node_debug.py)
- [node_debug_vo.py](file://core/workflow/domain/entities/node_debug_vo.py)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py)
</cite>

## 目录
1. [简介](#简介)
2. [调试API端点](#调试api端点)
3. [请求参数](#请求参数)
4. [响应结构](#响应结构)
5. [调试模式参数](#调试模式参数)
6. [执行痕迹与状态监控](#执行痕迹与状态监控)
7. [性能指标](#性能指标)
8. [API集成](#api集成)
9. [使用示例](#使用示例)
10. [最佳实践](#最佳实践)

## 简介
调试API为工作流代理提供了详细的调试功能，允许开发者在开发和问题排查过程中获取节点级执行痕迹、变量状态和性能指标。该API通过POST请求与/v1/workflow_agent/debug端点交互，支持详细的调试信息收集和分析。

**Section sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)

## 调试API端点
调试API提供了一个主要的调试端点，用于执行工作流代理的调试操作。

```mermaid
flowchart TD
Client["客户端应用"] --> |POST /v1/workflow_agent/debug| API["调试API"]
API --> |验证请求| Auth["认证服务"]
API --> |处理调试请求| Runner["WorkflowAgentRunner"]
Runner --> |收集执行痕迹| Trace["节点痕迹收集器"]
Runner --> |监控性能指标| Metrics["性能指标服务"]
API --> |返回调试结果| Client
```

**Diagram sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)

**Section sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)

## 请求参数
调试API接受POST请求，包含必要的认证信息和调试参数。

### HTTP方法和URL模式
- **HTTP方法**: POST
- **URL模式**: /v1/workflow_agent/debug

### 认证方法
API使用基于Header的认证，需要在请求头中包含用户身份信息：
- `x_consumer_username`: 消费者用户名，用于身份验证

### 主要请求参数
```mermaid
classDiagram
class CustomCompletionInputs {
+model_config_inputs : CustomCompletionModelConfigInputs
+instruction : CustomCompletionInstructionInputs
+plugin : CustomCompletionPluginInputs
+max_loop_count : int
}
class CustomCompletionModelConfigInputs {
+domain : str
+api : str
+api_key : str
}
class CustomCompletionInstructionInputs {
+reasoning : str
+answer : str
}
class CustomCompletionPluginInputs {
+tools : list[str | dict]
+mcp_server_ids : list[str]
+mcp_server_urls : list[str]
+workflow_ids : list[str]
+knowledge : list[CustomCompletionPluginKnowledgeInputs]
}
class CustomCompletionPluginKnowledgeInputs {
+name : str
+description : str
+top_k : int
+match : CustomCompletionPluginKnowledgeMatchInputs
+repo_type : int
}
class CustomCompletionPluginKnowledgeMatchInputs {
+repo_ids : list[str]
+doc_ids : list[str]
}
CustomCompletionInputs --> CustomCompletionModelConfigInputs : "包含"
CustomCompletionInputs --> CustomCompletionInstructionInputs : "包含"
CustomCompletionInputs --> CustomCompletionPluginInputs : "包含"
CustomCompletionPluginInputs --> CustomCompletionPluginKnowledgeInputs : "包含"
CustomCompletionPluginKnowledgeInputs --> CustomCompletionPluginKnowledgeMatchInputs : "包含"
```

**Diagram sources**
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py#L1-L51)
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)

**Section sources**
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py#L1-L51)
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)

## 响应结构
调试API返回流式响应，包含详细的执行信息和调试数据。

### 响应格式
API返回StreamingResponse，媒体类型为application/json，包含以下头部信息：
- Cache-Control: no-cache
- X-Accel-Buffering: no

### 响应数据结构
```mermaid
classDiagram
class ReasonChatCompletionChunk {
+code : int
+message : str
+id : str
+choices : list
+created : int
+model : str
+object : str
}
class NodeTracePatch {
+trace : list[NodeLog]
+start_time : int
+record_start() : void
+record_end() : void
+upload(status, log_caller, span) : dict
}
class NodeTraceLog {
+set_status(code, message) : void
+set_end() : void
+model_dump() : dict
}
class NodeLog {
+node_id : str
+status : str
+start_time : int
+end_time : int
+duration : int
+variables : dict
+inputs : dict
+outputs : dict
}
NodeTracePatch --> NodeTraceLog : "继承"
NodeTracePatch --> NodeLog : "包含"
ReasonChatCompletionChunk --> NodeTracePatch : "可能包含"
```

**Diagram sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)
- [completion_chunk.py](file://core/agent/api/schemas/completion_chunk.py#L1-L20)

**Section sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)

## 调试模式参数
在调试模式下，API支持额外的请求参数，用于控制调试行为和信息收集级别。

### 调试信息参数
- **debug_info**: 布尔值，指示是否返回详细的调试信息
- **trace_level**: 枚举值，指定痕迹收集的详细程度
  - BASIC: 基本执行路径信息
  - DETAILED: 详细节点执行信息
  - FULL: 完整的执行痕迹，包括变量状态

### 调试配置参数
```mermaid
flowchart TD
Start["调试配置开始"] --> TraceLevel{"trace_level"}
TraceLevel --> |BASIC| BasicTrace["收集基本执行路径"]
TraceLevel --> |DETAILED| DetailedTrace["收集详细节点信息"]
TraceLevel --> |FULL| FullTrace["收集完整执行痕迹"]
BasicTrace --> VariableState{"debug_variables?"}
DetailedTrace --> VariableState
FullTrace --> VariableState
VariableState --> |是| CollectVariables["收集变量状态"]
VariableState --> |否| SkipVariables["跳过变量状态"]
CollectVariables --> PerformanceMetrics["收集性能指标"]
SkipVariables --> PerformanceMetrics
PerformanceMetrics --> End["调试配置完成"]
```

**Diagram sources**
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py#L1-L51)
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)

**Section sources**
- [workflow_agent_inputs.py](file://core/agent/api/schemas/workflow_agent_inputs.py#L1-L51)
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)

## 执行痕迹与状态监控
调试API提供了详细的节点级执行痕迹和变量状态监控功能。

### 节点执行痕迹
```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "调试API"
participant Runner as "WorkflowAgentRunner"
participant Node as "工作流节点"
participant Trace as "痕迹收集器"
Client->>API : POST /v1/workflow_agent/debug
API->>Runner : 初始化执行器
Runner->>Node : 执行节点1
Node->>Trace : 记录开始时间
Node->>Node : 处理逻辑
Node->>Trace : 记录结束时间
Trace->>Runner : 返回节点痕迹
Runner->>Node : 执行节点2
Node->>Trace : 记录开始时间
Node->>Node : 处理逻辑
Node->>Trace : 记录结束时间
Trace->>Runner : 返回节点痕迹
Runner->>API : 汇总所有痕迹
API->>Client : 流式返回调试结果
Note over Trace,Runner : 收集每个节点的<br/>执行时间、状态和结果
```

**Diagram sources**
- [workflow_agent_runner.py](file://core/agent/engine/workflow_agent_runner.py#L1-L6)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)
- [node_debug.py](file://core/workflow/api/v1/chat/node_debug.py#L1-L30)

**Section sources**
- [workflow_agent_runner.py](file://core/agent/engine/workflow_agent_runner.py#L1-L6)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)

## 性能指标
调试API收集和报告工作流执行的性能指标，帮助优化工作流性能。

### 性能指标类型
```mermaid
erDiagram
PERFORMANCE_METRICS {
string metric_id PK
string workflow_id
string node_id
string metric_type
float value
timestamp timestamp
string unit
json metadata
}
EXECUTION_TRACE {
string trace_id PK
string workflow_id
string status
timestamp start_time
timestamp end_time
int duration_ms
int node_count
int error_count
}
VARIABLE_STATE {
string state_id PK
string trace_id
string variable_name
string variable_type
json value
timestamp timestamp
}
PERFORMANCE_METRICS ||--o{ EXECUTION_TRACE : "属于"
VARIABLE_STATE ||--o{ EXECUTION_TRACE : "属于"
```

**Diagram sources**
- [node_debug_vo.py](file://core/workflow/domain/entities/node_debug_vo.py#L1-L40)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)

**Section sources**
- [node_debug_vo.py](file://core/workflow/domain/entities/node_debug_vo.py#L1-L40)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)

## API集成
调试API与workflow_agent_runner的调试功能紧密集成，提供完整的调试解决方案。

### 集成架构
```mermaid
graph TB
subgraph "前端界面"
DebuggerUI["调试UI"]
DebuggerUI --> |发送调试请求| API
end
subgraph "后端服务"
API["调试API"]
API --> |创建| Runner["WorkflowAgentRunner"]
Runner --> |执行| Nodes["工作流节点"]
Runner --> |收集| Trace["节点痕迹"]
Runner --> |监控| Metrics["性能指标"]
Trace --> |存储| TraceDB["痕迹数据库"]
Metrics --> |存储| MetricsDB["指标数据库"]
API --> |返回| DebuggerUI
end
subgraph "数据存储"
TraceDB["痕迹数据库"]
MetricsDB["指标数据库"]
end
```

**Diagram sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [workflow_agent_runner.py](file://core/agent/engine/workflow_agent_runner.py#L1-L6)
- [app.py](file://core/agent/api/app.py#L1-L85)

**Section sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [workflow_agent_runner.py](file://core/agent/engine/workflow_agent_runner.py#L1-L6)

## 使用示例
以下示例展示了如何使用调试API进行工作流调试。

### 基本调试请求
```json
{
  "model_config": {
    "domain": "general",
    "api": "llm_api",
    "api_key": "your_api_key"
  },
  "instruction": {
    "reasoning": "analyze the input and provide a response",
    "answer": "generate a comprehensive answer"
  },
  "plugin": {
    "tools": [],
    "mcp_server_ids": [],
    "mcp_server_urls": [],
    "workflow_ids": [],
    "knowledge": []
  },
  "max_loop_count": 10
}
```

### 调试响应示例
```json
{
  "code": 200,
  "message": "success",
  "id": "trace_12345",
  "choices": [
    {
      "node_id": "node_1",
      "status": "success",
      "start_time": 1700000000000,
      "end_time": 1700000001000,
      "duration": 1000,
      "inputs": {
        "question": "What is the weather today?"
      },
      "outputs": {
        "response": "The weather is sunny today."
      },
      "variables": {
        "temperature": 25,
        "condition": "sunny"
      }
    }
  ],
  "created": 1700000000,
  "model": "workflow_agent",
  "object": "chat.completion.chunk"
}
```

**Section sources**
- [workflow_agent.py](file://core/agent/api/v1/workflow_agent.py#L1-L106)
- [node_trace_patch.py](file://core/agent/api/schemas/node_trace_patch.py#L1-L51)

## 最佳实践
使用调试API时，遵循以下最佳实践可以提高调试效率和工作流性能。

### 调试场景
```mermaid
flowchart TD
A["调试场景"] --> B["问题排查"]
A --> C["性能优化"]
A --> D["功能验证"]
B --> B1["识别失败节点"]
B --> B2["分析错误原因"]
B --> B3["修复逻辑错误"]
C --> C1["识别性能瓶颈"]
C --> C2["优化慢速节点"]
C --> C3["减少执行时间"]
D --> D1["验证节点输出"]
D --> D2["检查变量状态"]
D --> D3["确认工作流逻辑"]
style A fill:#f9f,stroke:#333,stroke-width:2px
```

**Diagram sources**
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)
- [node_debug.py](file://core/workflow/api/v1/chat/node_debug.py#L1-L30)

**Section sources**
- [debug.py](file://core/workflow/api/v1/chat/debug.py#L1-L50)
- [node_debug.py](file://core/workflow/api/v1/chat/node_debug.py#L1-L30)