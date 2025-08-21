# Autonomous Agentic Workflow Service Architecture

## Executive Summary

This document outlines the comprehensive architecture for an autonomous agentic workflow service that orchestrates multi-agent processes with human-in-the-loop capabilities, conditional branching, and robust execution management.

## 1. Existing Services Analysis & Integration Points

### 1.1 Current Service Landscape

```mermaid
graph TB
    subgraph "Existing Services"
        LLM[LLM Service<br/>OpenAI-compatible API]
        AGENT[Agent Service<br/>4 Tool Types + Execution]
        STATE[State Management Service<br/>Workspace/Play/Block State]
    end
    
    LLM --> AGENT
    AGENT --> STATE
    
    subgraph "Tool Types"
        API[API Tools]
        SCRIPT[Script Tools]
        UC[Use Case Tools]
        MCP[MCP Tools]
    end
    
    AGENT --> API
    AGENT --> SCRIPT
    AGENT --> UC
    AGENT --> MCP
```

### 1.2 Integration Requirements

**LLM Service Integration:**
- Workflow template generation using LLM capabilities
- Natural language workflow description parsing
- Agent capability analysis for optimal workflow suggestions

**Agent Service Integration:**
- Agent discovery and capability querying
- Agent execution orchestration
- Tool availability validation

**State Management Service Integration:**
- Workflow execution state persistence
- Block execution tracking
- Recovery state management
- Workspace context sharing

## 2. High-Level System Architecture

```mermaid
graph TB
    subgraph "Workflow Service Core"
        WTE[Workflow Template Engine]
        WEE[Workflow Execution Engine]
        DEP[Dependency Manager]
        HIL[Human-in-Loop Manager]
        REC[Recovery Manager]
    end
    
    subgraph "Data Layer"
        MONGO[MongoDB<br/>Workflow Templates]
        STATE_SVC[State Management Service<br/>Execution Data]
    end
    
    subgraph "External Services"
        LLM_SVC[LLM Service]
        AGENT_SVC[Agent Service]
    end
    
    subgraph "Interfaces"
        API[REST API]
        WS[WebSocket API]
        QUEUE[Message Queue]
    end
    
    API --> WTE
    API --> WEE
    WS --> HIL
    QUEUE --> HIL
    
    WTE --> MONGO
    WTE --> LLM_SVC
    WTE --> AGENT_SVC
    
    WEE --> DEP
    WEE --> HIL
    WEE --> REC
    WEE --> STATE_SVC
    WEE --> AGENT_SVC
    
    DEP --> STATE_SVC
    REC --> STATE_SVC
```

## 3. Core Components

### 3.1 Workflow Template Engine
**Responsibilities:**
- Autonomous workflow template generation
- Template validation and optimization
- Template versioning and storage
- Agent capability analysis integration

**Key Features:**
- Natural language goal parsing
- Agent-to-block mapping optimization
- Dependency graph generation
- Template reusability analysis

### 3.2 Workflow Execution Engine
**Responsibilities:**
- Workflow instance management
- Block execution orchestration
- Parallel execution coordination
- Conditional branching logic

**Key Features:**
- Dynamic execution planning
- Resource allocation
- Progress tracking
- Error handling and propagation

### 3.3 Dependency Manager
**Responsibilities:**
- Dependency graph resolution
- Parallel execution opportunity identification
- Blocking condition management
- Circular dependency detection

### 3.4 Human-in-Loop Manager
**Responsibilities:**
- Approval workflow management
- Real-time notification handling
- Asynchronous approval queuing
- Input collection and validation

### 3.5 Recovery Manager
**Responsibilities:**
- Failure detection and classification
- Restart strategy determination
- State recovery coordination
- Partial execution preservation

## 4. Data Models

### 4.1 Workflow Template Schema (MongoDB)

```json
{
  "_id": "ObjectId",
  "templateId": "string",
  "name": "string",
  "description": "string",
  "version": "string",
  "createdAt": "Date",
  "updatedAt": "Date",
  "createdBy": "string",
  "tags": ["string"],
  "metadata": {
    "estimatedDuration": "number",
    "complexity": "string",
    "category": "string"
  },
  "blocks": [
    {
      "blockId": "string",
      "name": "string",
      "description": "string",
      "agentId": "string",
      "agentType": "string",
      "toolType": "string",
      "configuration": "object",
      "dependencies": ["string"],
      "conditions": {
        "type": "string",
        "expression": "string",
        "branches": [
          {
            "condition": "string",
            "nextBlocks": ["string"]
          }
        ]
      },
      "humanInLoop": {
        "required": "boolean",
        "type": "string",
        "approvalLevel": "string",
        "inputSchema": "object"
      },
      "retryPolicy": {
        "maxRetries": "number",
        "backoffStrategy": "string",
        "retryConditions": ["string"]
      }
    }
  ],
  "globalSettings": {
    "timeout": "number",
    "maxParallelBlocks": "number",
    "defaultRetryPolicy": "object",
    "humanInLoopDefaults": "object"
  }
}
```

### 4.2 Workflow Execution Model (State Management Service)

```json
{
  "executionId": "string",
  "templateId": "string",
  "templateVersion": "string",
  "workspaceId": "string",
  "status": "string",
  "startedAt": "Date",
  "completedAt": "Date",
  "createdBy": "string",
  "currentBlocks": ["string"],
  "completedBlocks": ["string"],
  "failedBlocks": ["string"],
  "blockedBlocks": ["string"],
  "blockExecutions": [
    {
      "blockId": "string",
      "executionId": "string",
      "agentId": "string",
      "status": "string",
      "startedAt": "Date",
      "completedAt": "Date",
      "input": "object",
      "output": "object",
      "error": "object",
      "retryCount": "number",
      "humanInteractions": [
        {
          "type": "string",
          "requestedAt": "Date",
          "respondedAt": "Date",
          "response": "object",
          "approver": "string"
        }
      ]
    }
  ],
  "globalContext": "object",
  "recoveryPoints": [
    {
      "timestamp": "Date",
      "state": "object",
      "completedBlocks": ["string"]
    }
  ]
}
```

## 5. API Design

### 5.1 Template Management APIs

```
POST /api/v1/templates/generate
- Generate workflow template from natural language description
- Body: { goal: string, constraints?: object, preferences?: object }
- Response: { templateId: string, template: WorkflowTemplate }

GET /api/v1/templates
- List workflow templates with filtering
- Query: { category?, tags?, search?, limit?, offset? }
- Response: { templates: WorkflowTemplate[], total: number }

GET /api/v1/templates/{templateId}
- Get specific template
- Response: WorkflowTemplate

PUT /api/v1/templates/{templateId}
- Update template
- Body: WorkflowTemplate
- Response: WorkflowTemplate

DELETE /api/v1/templates/{templateId}
- Delete template
- Response: { success: boolean }

POST /api/v1/templates/{templateId}/validate
- Validate template against current agent capabilities
- Response: { valid: boolean, issues: ValidationIssue[] }
```

### 5.2 Execution Management APIs

```
POST /api/v1/executions
- Start workflow execution
- Body: { templateId: string, input?: object, workspaceId: string }
- Response: { executionId: string, status: string }

GET /api/v1/executions/{executionId}
- Get execution status and details
- Response: WorkflowExecution

POST /api/v1/executions/{executionId}/pause
- Pause execution
- Response: { success: boolean }

POST /api/v1/executions/{executionId}/resume
- Resume paused execution
- Response: { success: boolean }

POST /api/v1/executions/{executionId}/restart
- Restart failed execution
- Body: { fromBlock?: string, resetState?: boolean }
- Response: { success: boolean }

POST /api/v1/executions/{executionId}/cancel
- Cancel execution
- Response: { success: boolean }

GET /api/v1/executions/{executionId}/logs
- Get execution logs
- Query: { blockId?, level?, limit?, offset? }
- Response: { logs: LogEntry[] }
```

### 5.3 Human-in-Loop APIs

```
GET /api/v1/approvals/pending
- Get pending approvals for user
- Query: { executionId?, blockId?, type? }
- Response: { approvals: PendingApproval[] }

POST /api/v1/approvals/{approvalId}/respond
- Respond to approval request
- Body: { approved: boolean, response?: object, comments?: string }
- Response: { success: boolean }

WebSocket: /ws/approvals
- Real-time approval notifications
- Events: approval_requested, approval_responded, approval_timeout
```

## 6. Integration Patterns

### 6.1 Agent Service Integration

```mermaid
sequenceDiagram
    participant WS as Workflow Service
    participant AS as Agent Service
    participant SM as State Management
    
    WS->>AS: Query available agents
    AS->>WS: Return agent capabilities
    WS->>AS: Execute block with agent
    AS->>WS: Execution started
    AS->>SM: Update block state
    AS->>WS: Execution completed
    WS->>SM: Update workflow state
```

### 6.2 LLM Service Integration

```mermaid
sequenceDiagram
    participant WS as Workflow Service
    participant LLM as LLM Service
    participant AS as Agent Service
    
    WS->>LLM: Generate workflow from goal
    LLM->>WS: Return workflow structure
    WS->>AS: Validate agent assignments
    AS->>WS: Return validation results
    WS->>LLM: Optimize workflow based on validation
    LLM->>WS: Return optimized workflow
```

### 6.3 State Management Integration

```mermaid
sequenceDiagram
    participant WS as Workflow Service
    participant SM as State Management
    participant AS as Agent Service
    
    WS->>SM: Create execution state
    SM->>WS: Return execution ID
    WS->>AS: Execute block
    AS->>SM: Update block state
    SM->>WS: Notify state change
    WS->>SM: Query execution state
    SM->>WS: Return current state
```

## 7. Execution Flow Scenarios

### 7.1 Linear Workflow Execution

```mermaid
flowchart TD
    START([Start Execution]) --> LOAD[Load Template]
    LOAD --> VALIDATE[Validate Dependencies]
    VALIDATE --> EXEC1[Execute Block 1]
    EXEC1 --> CHECK1{Success?}
    CHECK1 -->|Yes| EXEC2[Execute Block 2]
    CHECK1 -->|No| RETRY1[Retry Block 1]
    RETRY1 --> CHECK1
    EXEC2 --> CHECK2{Success?}
    CHECK2 -->|Yes| EXEC3[Execute Block 3]
    CHECK2 -->|No| RETRY2[Retry Block 2]
    RETRY2 --> CHECK2
    EXEC3 --> END([Complete])
```

### 7.2 Parallel Execution Flow

```mermaid
flowchart TD
    START([Start Execution]) --> LOAD[Load Template]
    LOAD --> ANALYZE[Analyze Dependencies]
    ANALYZE --> EXEC1[Execute Block 1]
    EXEC1 --> PARALLEL{Parallel Blocks Available?}
    PARALLEL -->|Yes| EXEC2A[Execute Block 2A]
    PARALLEL -->|Yes| EXEC2B[Execute Block 2B]
    PARALLEL -->|Yes| EXEC2C[Execute Block 2C]
    EXEC2A --> SYNC[Synchronization Point]
    EXEC2B --> SYNC
    EXEC2C --> SYNC
    SYNC --> EXEC3[Execute Block 3]
    EXEC3 --> END([Complete])
```

### 7.3 Human-in-Loop Flow

```mermaid
flowchart TD
    START([Block Execution]) --> CHECK{Human Approval Required?}
    CHECK -->|No| EXECUTE[Execute Block]
    CHECK -->|Yes| NOTIFY[Send Approval Request]
    NOTIFY --> WAIT[Wait for Response]
    WAIT --> TIMEOUT{Timeout?}
    TIMEOUT -->|Yes| ESCALATE[Escalate Request]
    TIMEOUT -->|No| RESPONSE{Approved?}
    ESCALATE --> WAIT
    RESPONSE -->|Yes| EXECUTE
    RESPONSE -->|No| REJECT[Mark Block Rejected]
    EXECUTE --> SUCCESS{Success?}
    SUCCESS -->|Yes| COMPLETE[Mark Complete]
    SUCCESS -->|No| FAIL[Mark Failed]
    REJECT --> END([End])
    COMPLETE --> END
    FAIL --> END
```

### 7.4 Conditional Branching Flow

```mermaid
flowchart TD
    START([Execute Block]) --> COMPLETE[Block Completed]
    COMPLETE --> EVAL[Evaluate Conditions]
    EVAL --> COND1{Condition 1?}
    COND1 -->|True| BRANCH1[Execute Branch 1 Blocks]
    COND1 -->|False| COND2{Condition 2?}
    COND2 -->|True| BRANCH2[Execute Branch 2 Blocks]
    COND2 -->|False| DEFAULT[Execute Default Branch]
    BRANCH1 --> MERGE[Merge Point]
    BRANCH2 --> MERGE
    DEFAULT --> MERGE
    MERGE --> NEXT[Continue Workflow]
    NEXT --> END([End])
```

### 7.5 Recovery and Restart Flow

```mermaid
flowchart TD
    START([Failure Detected]) --> ANALYZE[Analyze Failure Type]
    ANALYZE --> TYPE{Failure Type}
    TYPE -->|Transient| RETRY[Retry Block]
    TYPE -->|Agent Error| REASSIGN[Reassign to Different Agent]
    TYPE -->|Configuration| HUMAN[Request Human Intervention]
    TYPE -->|Critical| FAIL[Mark Workflow Failed]
    
    RETRY --> SUCCESS1{Success?}
    SUCCESS1 -->|Yes| CONTINUE[Continue Execution]
    SUCCESS1 -->|No| ESCALATE1[Escalate Failure]
    
    REASSIGN --> SUCCESS2{Success?}
    SUCCESS2 -->|Yes| CONTINUE
    SUCCESS2 -->|No| ESCALATE2[Escalate Failure]
    
    HUMAN --> RESOLVED{Resolved?}
    RESOLVED -->|Yes| RESTART[Restart from Recovery Point]
    RESOLVED -->|No| FAIL
    
    RESTART --> CONTINUE
    ESCALATE1 --> HUMAN
    ESCALATE2 --> HUMAN
    CONTINUE --> END([Resume Normal Flow])
    FAIL --> CLEANUP[Cleanup Resources]
    CLEANUP --> NOTIFY[Notify Stakeholders]
    NOTIFY --> FINAL([End])
```

## 8. Database Schema Design

### 8.1 MongoDB Collections

**Templates Collection:**
```javascript
// Indexes
db.templates.createIndex({ "templateId": 1 }, { unique: true })
db.templates.createIndex({ "tags": 1 })
db.templates.create
