# Workflow API Specifications

## Overview

This document defines the REST API specifications for the Workflow Service, providing endpoints for workflow creation, execution, monitoring, and management.

## Base Configuration

```yaml
openapi: 3.1.0
info:
  title: Workflow Service API
  version: 1.0.0
  description: API for creating and managing autonomous multi-agent workflows
  contact:
    name: Workflow Service Team
    email: workflow-service@lionis.ai

servers:
  - url: https://dev.lionis.ai/api/v1/workflows
    description: Development server
  - url: https://api.lionis.ai/api/v1/workflows
    description: Production server

security:
  - bearerAuth: []

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

## Core API Endpoints

### Workflow Management

#### Create Workflow from Natural Language

```yaml
/workflows:
  post:
    summary: Create workflow from natural language description
    description: Converts a natural language description into an executable workflow
    tags:
      - Workflow Management
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required:
              - description
            properties:
              description:
                type: string
                description: Natural language description of the workflow
                example: "Create a customer onboarding process that verifies identity, sets up accounts, and sends welcome emails"
              context:
                $ref: '#/components/schemas/BusinessContext'
              executionMode:
                $ref: '#/components/schemas/ExecutionMode'
              templateHints:
                type: array
                items:
                  type: string
                description: Suggested template categories or IDs
              customizations:
                type: object
                description: Custom parameters or modifications
    responses:
      '201':
        description: Workflow created successfully
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WorkflowCreationResponse'
      '400':
        description: Invalid request or validation errors
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ErrorResponse'
      '401':
        description: Unauthorized
      '429':
        description: Rate limit exceeded
```

#### Create Workflow from Template

```yaml
/workflows/from-template:
  post:
    summary: Create workflow from template
    description: Instantiates a workflow from a predefined template
    tags:
      - Workflow Management
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required:
              - templateId
              - parameters
            properties:
              templateId:
                type: string
                description: ID of the template to instantiate
                example: "customer-onboarding-v1"
              parameters:
                type: object
                description: Template parameters
                example:
                  verification_method: "document_upload"
                  account_type: "premium"
                  followup_delay_days: 7
              customizations:
                type: array
                items:
                  $ref: '#/components/schemas/TemplateCustomization'
              name:
                type: string
                description: Custom name for the workflow instance
    responses:
      '201':
        description: Workflow created from template
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WorkflowDefinition'
      '400':
        description: Invalid template parameters
      '404':
        description: Template not found
```

#### List Workflows

```yaml
/workflows:
  get:
    summary: List workflows
    description: Retrieve a list of workflows with filtering and pagination
    tags:
      - Workflow Management
    parameters:
      - name: status
        in: query
        schema:
          type: string
          enum: [DRAFT, ACTIVE, COMPLETED, FAILED, PAUSED]
        description: Filter by workflow status
      - name: category
        in: query
        schema:
          type: string
        description: Filter by workflow category
      - name: createdBy
        in: query
        schema:
          type: string
        description: Filter by creator
      - name: page
        in: query
        schema:
          type: integer
          minimum: 1
          default: 1
        description: Page number
      - name: limit
        in: query
        schema:
          type: integer
          minimum: 1
          maximum: 100
          default: 20
        description: Number of items per page
      - name: sortBy
        in: query
        schema:
          type: string
          enum: [createdAt, name, status, complexity]
          default: createdAt
        description: Sort field
      - name: sortOrder
        in: query
        schema:
          type: string
          enum: [asc, desc]
          default: desc
        description: Sort order
    responses:
      '200':
        description: List of workflows
        content:
          application/json:
            schema:
              type: object
              properties:
                workflows:
                  type: array
                  items:
                    $ref: '#/components/schemas/WorkflowSummary'
                pagination:
                  $ref: '#/components/schemas/PaginationInfo'
```

### Workflow Execution

#### Execute Workflow

```yaml
/workflows/{workflowId}/execute:
  post:
    summary: Execute workflow
    description: Start execution of a workflow with provided inputs
    tags:
      - Workflow Execution
    parameters:
      - name: workflowId
        in: path
        required: true
        schema:
          type: string
        description: Workflow ID
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required:
              - inputs
            properties:
              inputs:
                type: object
                description: Workflow input parameters
                example:
                  customer_data:
                    name: "John Doe"
                    email: "john@example.com"
                    phone: "+1234567890"
              executionOptions:
                $ref: '#/components/schemas/ExecutionOptions'
              approvalSettings:
                $ref: '#/components/schemas/ApprovalSettings'
    responses:
      '202':
        description: Execution started
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WorkflowExecution'
      '400':
        description: Invalid inputs or workflow not ready
      '404':
        description: Workflow not found
      '409':
        description: Workflow already executing
```

#### Get Execution Status

```yaml
/executions/{executionId}:
  get:
    summary: Get execution status
    description: Retrieve detailed status of a workflow execution
    tags:
      - Workflow Execution
    parameters:
      - name: executionId
        in: path
        required: true
        schema:
          type: string
        description: Execution ID
      - name: includeTaskDetails
        in: query
        schema:
          type: boolean
          default: false
        description: Include detailed task execution information
    responses:
      '200':
        description: Execution details
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WorkflowExecutionDetails'
      '404':
        description: Execution not found
```

#### Control Execution

```yaml
/executions/{executionId}/control:
  post:
    summary: Control workflow execution
    description: Pause, resume, or cancel workflow execution
    tags:
      - Workflow Execution
    parameters:
      - name: executionId
        in: path
        required: true
        schema:
          type: string
        description: Execution ID
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required:
              - action
            properties:
              action:
                type: string
                enum: [PAUSE, RESUME, CANCEL, RETRY_FAILED]
                description: Control action to perform
              reason:
                type: string
                description: Reason for the action
              retryOptions:
                type: object
                description: Options for retry action
                properties:
                  taskIds:
                    type: array
                    items:
                      type: string
                    description: Specific tasks to retry (if not provided, retries all failed tasks)
    responses:
      '200':
        description: Control action executed
        content:
          application/json:
            schema:
              type: object
              properties:
                status:
                  type: string
                message:
                  type: string
                executionStatus:
                  $ref: '#/components/schemas/ExecutionStatus'
      '400':
        description: Invalid action for current execution state
      '404':
        description: Execution not found
```

### Template Management

#### List Templates

```yaml
/templates:
  get:
    summary: List workflow templates
    description: Retrieve available workflow templates
    tags:
      - Template Management
    parameters:
      - name: category
        in: query
        schema:
          $ref: '#/components/schemas/TemplateCategory'
        description: Filter by template category
      - name: complexity
        in: query
        schema:
          $ref: '#/components/schemas/ComplexityLevel'
        description: Filter by complexity level
      - name: tags
        in: query
        schema:
          type: array
          items:
            type: string
        description: Filter by tags
      - name: search
        in: query
        schema:
          type: string
        description: Search in template name and description
    responses:
      '200':
        description: List of templates
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: '#/components/schemas/WorkflowTemplate'
```

#### Get Template Details

```yaml
/templates/{templateId}:
  get:
    summary: Get template details
    description: Retrieve detailed information about a specific template
    tags:
      - Template Management
    parameters:
      - name: templateId
        in: path
        required: true
        schema:
          type: string
        description: Template ID
    responses:
      '200':
        description: Template details
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WorkflowTemplateDetails'
      '404':
        description: Template not found
```

### Monitoring and Analytics

#### Get Execution Metrics

```yaml
/executions/{executionId}/metrics:
  get:
    summary: Get execution metrics
    description: Retrieve performance metrics for a workflow execution
    tags:
      - Monitoring
    parameters:
      - name: executionId
        in: path
        required: true
        schema:
          type: string
        description: Execution ID
    responses:
      '200':
        description: Execution metrics
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ExecutionMetrics'
```

#### Get Agent Activity

```yaml
/executions/{executionId}/agents:
  get:
    summary: Get agent activity
    description: Retrieve information about agents involved in execution
    tags:
      - Monitoring
    parameters:
      - name: executionId
        in: path
        required: true
        schema:
          type: string
        description: Execution ID
    responses:
      '200':
        description: Agent activity information
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: '#/components/schemas/AgentActivity'
```

### Approval Management

#### Get Pending Approvals

```yaml
/approvals/pending:
  get:
    summary: Get pending approvals
    description: Retrieve list of tasks waiting for approval
    tags:
      - Approval Management
    parameters:
      - name: userId
        in: query
        schema:
          type: string
        description: Filter by assigned approver
      - name: priority
        in: query
        schema:
          type: string
          enum: [LOW, MEDIUM, HIGH, CRITICAL]
        description: Filter by priority level
    responses:
      '200':
        description: List of pending approvals
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: '#/components/schemas/PendingApproval'
```

#### Submit Approval Decision

```yaml
/approvals/{approvalId}/decision:
  post:
    summary: Submit approval decision
    description: Approve or reject a pending task
    tags:
      - Approval Management
    parameters:
      - name: approvalId
        in: path
        required: true
        schema:
          type: string
        description: Approval ID
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required:
              - decision
            properties:
              decision:
                type: string
                enum: [APPROVED, REJECTED]
                description: Approval decision
              comments:
                type: string
                description: Comments or feedback
              modifications:
                type: object
                description: Suggested modifications to task parameters
    responses:
      '200':
        description: Decision recorded
        content:
          application/json:
            schema:
              type: object
              properties:
                status:
                  type: string
                message:
                  type: string
                nextAction:
                  type: string
                  description: What happens next in the workflow
```

## Data Schemas

### Core Workflow Schemas

```yaml
components:
  schemas:
    WorkflowDefinition:
      type: object
      required:
        - id
        - name
        - description
        - tasks
        - dependencies
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
          maxLength: 255
        description:
          type: string
          maxLength: 1000
        version:
          type: string
          pattern: '^[0-9]+\.[0-9]+\.[0-9]+$'
        createdAt:
          type: string
          format: date-time
        createdBy:
          type: string
        tasks:
          type: array
          items:
            $ref: '#/components/schemas/WorkflowTask'
        dependencies:
          type: array
          items:
            $ref: '#/components/schemas/TaskDependency'
        executionMode:
          $ref: '#/components/schemas/ExecutionMode'
        approvalPoints:
          type: array
          items:
            $ref: '#/components/schemas/ApprovalPoint'
        requiredAgentTypes:
          type: array
          items:
            $ref: '#/components/schemas/AgentTypeRequirement'
        estimatedDuration:
          type: integer
          description: Estimated duration in milliseconds
        complexity:
          $ref: '#/components/schemas/ComplexityLevel'

    WorkflowTask:
      type: object
      required:
        - id
        - name
        - type
        - agentType
      properties:
        id:
          type: string
        name:
          type: string
        description:
          type: string
        type:
          $ref: '#/components/schemas/TaskType'
        agentType:
          type: string
        agentSelectionCriteria:
          $ref: '#/components/schemas/AgentSelectionCriteria'
        inputs:
          type: array
          items:
            $ref: '#/components/schemas/TaskInputDefinition'
        outputs:
          type: array
          items:
            $ref: '#/components/schemas/TaskOutputDefinition'
        timeout:
          type: integer
          description: Task timeout in milliseconds
        retryPolicy:
          $ref: '#/components/schemas/RetryPolicy'
        criticalityLevel:
          $ref: '#/components/schemas/CriticalityLevel'
        requiresApproval:
          type: boolean
          default: false
        approvalCriteria:
          $ref: '#/components/schemas/ApprovalCriteria'

    WorkflowExecution:
      type: object
      required:
        - id
        - workflowDefinitionId
        - status
        - startedAt
      properties:
        id:
          type: string
          format