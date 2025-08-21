# Workflow Execution Details - Continued

## Task Scheduling and Coordination (Continued)

```typescript
class ScheduleCoordinator {
  private async findSuitableAgent(taskDef: WorkflowTask): Promise<Agent | null> {
    // Query existing agent pools
    const availableAgents = await this.agentService.getAvailableAgents({
      agentType: taskDef.preferredAgentType,
      capabilities: taskDef.agentSelectionCriteria.requiredCapabilities,
      status: 'IDLE'
    });
    
    if (availableAgents.length === 0) {
      return null;
    }
    
    // Select best agent based on criteria
    return this.selectBestAgent(availableAgents, taskDef.agentSelectionCriteria);
  }
  
  private selectBestAgent(
    agents: Agent[], 
    criteria: AgentSelectionCriteria
  ): Agent {
    // Score agents based on selection criteria
    const scoredAgents = agents.map(agent => ({
      agent,
      score: this.calculateAgentScore(agent, criteria)
    }));
    
    // Return highest scoring agent
    return scoredAgents.sort((a, b) => b.score - a.score)[0].agent;
  }
  
  private async assignTaskToAgent(
    execution: WorkflowExecution,
    taskExecution: TaskExecution,
    agent: Agent
  ): Promise<void> {
    
    // Update task status
    taskExecution.status = TaskExecutionStatus.ASSIGNED;
    taskExecution.agentId = agent.id;
    
    // Prepare task inputs from workflow state
    const taskInputs = await this.prepareTaskInputs(execution, taskExecution);
    
    // Assign task to agent
    await this.agentService.assignTask(agent.id, {
      taskId: taskExecution.id,
      workspaceId: execution.workspaceId,
      playId: execution.playId,
      instructions: this.getTaskDefinition(execution, taskExecution.taskId).description,
      inputs: taskInputs,
      timeout: this.getTaskDefinition(execution, taskExecution.taskId).timeout
    });
    
    // Update execution state
    await this.updateExecutionState(execution);
  }
}
```

## State Management Integration

### Workspace and Play Integration

```typescript
interface WorkflowStateSchema {
  workspaceStructure: WorkspaceStructure;
  playConfiguration: PlayConfiguration;
  stateChannels: StateChannel[];
  coordinationPrimitives: CoordinationPrimitive[];
}

interface WorkspaceStructure {
  taskStates: TaskStateDefinition[];
  sharedData: SharedDataDefinition[];
  communicationChannels: CommunicationChannelDefinition[];
  checkpoints: CheckpointDefinition[];
}

interface PlayConfiguration {
  blocks: PlayBlockConfiguration[];
  progressTracking: ProgressTrackingConfiguration;
  synchronizationPoints: SynchronizationPoint[];
}

class StateIntegrationManager {
  async initializeWorkflowState(
    workspaceId: string,
    workflowDefinition: WorkflowDefinition,
    inputs: WorkflowInputs
  ): Promise<WorkflowState> {
    
    // Create task state entries
    await this.createTaskStates(workspaceId, workflowDefinition.tasks);
    
    // Initialize shared data
    await this.initializeSharedData(workspaceId, inputs);
    
    // Set up communication channels
    await this.createCommunicationChannels(workspaceId, workflowDefinition);
    
    // Create coordination primitives
    await this.createCoordinationPrimitives(workspaceId, workflowDefinition);
    
    return {
      workspaceId,
      initialized: true,
      version: 1,
      lastUpdated: new Date()
    };
  }
  
  private async createTaskStates(
    workspaceId: string,
    tasks: WorkflowTask[]
  ): Promise<void> {
    
    for (const task of tasks) {
      await this.stateService.createRecord({
        workspaceId,
        recordType: 'TASK_STATE',
        recordId: task.id,
        data: {
          taskId: task.id,
          status: 'PENDING',
          inputs: null,
          outputs: null,
          agentId: null,
          startedAt: null,
          completedAt: null,
          retryCount: 0
        }
      });
    }
  }
  
  private async initializeSharedData(
    workspaceId: string,
    inputs: WorkflowInputs
  ): Promise<void> {
    
    await this.stateService.createRecord({
      workspaceId,
      recordType: 'SHARED_DATA',
      recordId: 'workflow_inputs',
      data: inputs
    });
    
    await this.stateService.createRecord({
      workspaceId,
      recordType: 'SHARED_DATA',
      recordId: 'workflow_outputs',
      data: {}
    });
  }
  
  async updateTaskState(
    workspaceId: string,
    taskId: string,
    updates: Partial<TaskState>
  ): Promise<void> {
    
    await this.stateService.updateRecord({
      workspaceId,
      recordType: 'TASK_STATE',
      recordId: taskId,
      data: {
        ...updates,
        lastUpdated: new Date()
      }
    });
  }
  
  async getTaskState(workspaceId: string, taskId: string): Promise<TaskState> {
    const record = await this.stateService.getRecord({
      workspaceId,
      recordType: 'TASK_STATE',
      recordId: taskId
    });
    
    return record.data as TaskState;
  }
}
```

### Play Block Coordination

```typescript
class PlayBlockCoordinator {
  async createWorkflowBlocks(
    playId: string,
    workflowDefinition: WorkflowDefinition
  ): Promise<void> {
    
    // Create blocks for synchronization points
    const syncPoints = this.identifySynchronizationPoints(workflowDefinition);
    
    for (const syncPoint of syncPoints) {
      await this.stateService.createPlayBlock({
        playId,
        blockId: syncPoint.id,
        metadata: {
          type: syncPoint.type,
          dependentTasks: syncPoint.dependentTasks,
          condition: syncPoint.condition
        }
      });
    }
  }
  
  private identifySynchronizationPoints(
    workflowDefinition: WorkflowDefinition
  ): SynchronizationPoint[] {
    
    const syncPoints: SynchronizationPoint[] = [];
    
    // Find tasks with multiple dependencies (merge points)
    for (const task of workflowDefinition.tasks) {
      const dependencies = workflowDefinition.dependencies.filter(
        dep => dep.taskId === task.id
      );
      
      if (dependencies.length > 1) {
        syncPoints.push({
          id: `sync-${task.id}`,
          type: 'MERGE',
          dependentTasks: dependencies.flatMap(dep => dep.dependsOn),
          condition: 'ALL_COMPLETE'
        });
      }
    }
    
    // Find parallel execution points
    const parallelGroups = this.identifyParallelGroups(workflowDefinition);
    for (const group of parallelGroups) {
      syncPoints.push({
        id: `parallel-${group.id}`,
        type: 'PARALLEL_START',
        dependentTasks: group.tasks,
        condition: 'PARALLEL_EXECUTION'
      });
    }
    
    return syncPoints;
  }
  
  async waitForBlockCompletion(
    playId: string,
    blockId: string,
    timeout: number = 30000
  ): Promise<boolean> {
    
    const startTime = Date.now();
    
    while (Date.now() - startTime < timeout) {
      const block = await this.stateService.getPlayBlock(playId, blockId);
      
      if (block.status === 'COMPLETED') {
        return true;
      }
      
      // Wait before checking again
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    return false; // Timeout
  }
}
```

## Agent Coordination Mechanisms

### Agent Pool Management

```typescript
interface AgentPool {
  id: string;
  name: string;
  agentType: string;
  capabilities: string[];
  agents: PooledAgent[];
  configuration: PoolConfiguration;
}

interface PooledAgent {
  agentId: string;
  status: AgentStatus;
  currentTask?: string;
  lastActivity: Date;
  performance: AgentPerformance;
}

enum AgentStatus {
  IDLE = 'IDLE',
  BUSY = 'BUSY',
  MAINTENANCE = 'MAINTENANCE',
  OFFLINE = 'OFFLINE'
}

class AgentPoolManager {
  async getAvailableAgent(
    agentType: string,
    capabilities: string[],
    workspaceId: string
  ): Promise<PooledAgent | null> {
    
    const pool = await this.getAgentPool(agentType);
    
    if (!pool) {
      return null;
    }
    
    // Find idle agents with required capabilities
    const suitableAgents = pool.agents.filter(agent => 
      agent.status === AgentStatus.IDLE &&
      this.hasRequiredCapabilities(agent, capabilities)
    );
    
    if (suitableAgents.length === 0) {
      return null;
    }
    
    // Select best performing agent
    const selectedAgent = this.selectBestPerformingAgent(suitableAgents);
    
    // Reserve agent
    await this.reserveAgent(selectedAgent.agentId, workspaceId);
    
    return selectedAgent;
  }
  
  private selectBestPerformingAgent(agents: PooledAgent[]): PooledAgent {
    return agents.sort((a, b) => {
      // Sort by performance score (higher is better)
      const scoreA = this.calculatePerformanceScore(a.performance);
      const scoreB = this.calculatePerformanceScore(b.performance);
      return scoreB - scoreA;
    })[0];
  }
  
  private calculatePerformanceScore(performance: AgentPerformance): number {
    const successRate = performance.successfulTasks / performance.totalTasks;
    const avgDuration = performance.averageTaskDuration;
    const recentActivity = Date.now() - performance.lastTaskCompleted.getTime();
    
    // Weighted score considering success rate, speed, and recency
    return (successRate * 0.5) + 
           ((1 / avgDuration) * 0.3) + 
           ((1 / recentActivity) * 0.2);
  }
  
  async releaseAgent(agentId: string): Promise<void> {
    await this.updateAgentStatus(agentId, AgentStatus.IDLE);
    await this.updateAgentPerformance(agentId);
  }
}
```

### Task Assignment and Monitoring

```typescript
class TaskCoordinator {
  async assignTask(
    execution: WorkflowExecution,
    taskExecution: TaskExecution,
    agent: PooledAgent
  ): Promise<void> {
    
    // Update task execution status
    taskExecution.status = TaskExecutionStatus.ASSIGNED;
    taskExecution.agentId = agent.agentId;
    taskExecution.startedAt = new Date();
    
    // Prepare task context
    const taskContext = await this.prepareTaskContext(execution, taskExecution);
    
    // Assign to agent through Agent Service
    await this.agentService.assignTask(agent.agentId, {
      taskId: taskExecution.id,
      workspaceId: execution.workspaceId,
      playId: execution.playId,
      context: taskContext,
      timeout: this.getTaskTimeout(execution, taskExecution.taskId)
    });
    
    // Start monitoring
    await this.startTaskMonitoring(taskExecution);
  }
  
  private async prepareTaskContext(
    execution: WorkflowExecution,
    taskExecution: TaskExecution
  ): Promise<TaskContext> {
    
    const taskDef = this.getTaskDefinition(execution, taskExecution.taskId);
    
    // Gather inputs from workflow state
    const inputs = await this.gatherTaskInputs(execution, taskDef);
    
    // Prepare instructions
    const instructions = await this.generateTaskInstructions(taskDef, inputs);
    
    return {
      taskId: taskExecution.id,
      taskType: taskDef.type,
      instructions,
      inputs,
      expectedOutputs: taskDef.outputs,
      constraints: taskDef.parameters,
      criticalityLevel: taskDef.criticalityLevel
    };
  }
  
  private async gatherTaskInputs(
    execution: WorkflowExecution,
    taskDef: WorkflowTask
  ): Promise<any> {
    
    const inputs: any = {};
    
    for (const inputDef of taskDef.inputs) {
      switch (inputDef.source) {
        case 'WORKFLOW_INPUT':
          inputs[inputDef.name] = execution.inputs[inputDef.sourceKey];
          break;
          
        case 'TASK_OUTPUT':
          const taskOutput = await this.getTaskOutput(
            execution.workspaceId, 
            inputDef.sourceTaskId
          );
          inputs[inputDef.name] = taskOutput[inputDef.sourceKey];
          break;
          
        case 'SHARED_DATA':
          const sharedData = await this.getSharedData(
            execution.workspaceId, 
            inputDef.sourceKey
          );
          inputs[inputDef.name] = sharedData;
          break;
      }
    }
    
    return inputs;
  }
  
  async handleTaskCompletion(
    execution: WorkflowExecution,
    taskExecution: TaskExecution,
    result: TaskResult
  ): Promise<void> {
    
    // Update task execution
    taskExecution.status = result.success ? 
      TaskExecutionStatus.COMPLETED : 
      TaskExecutionStatus.FAILED;
    taskExecution.completedAt = new Date();
    taskExecution.outputs = result.outputs;
    taskExecution.error = result.error;
    
    // Update workflow state
    await this.updateTaskState(execution.workspaceId, taskExecution.taskId, {
      status: taskExecution.status,
      outputs: result.outputs,
      completedAt: taskExecution.completedAt,
      error: result.error
    });
    
    // Store outputs in shared data if needed
    if (result.success && result.outputs) {
      await this.storeTaskOutputs(execution, taskExecution, result.outputs);
    }
    
    // Release agent
    await this.agentPoolManager.releaseAgent(taskExecution.agentId!);
    
    // Check for dependent tasks
    await this.checkDependentTasks(execution, taskExecution.taskId);
    
    // Update workflow progress
    await this.updateWorkflowProgress(execution);
  }
}
```

## Error Handling and Recovery

### Error Classification and Response

```typescript
enum ErrorType {
  TASK_FAILURE = 'TASK_FAILURE',
  AGENT_UNAVAILABLE = 'AGENT_UNAVAILABLE',
  TIMEOUT = 'TIMEOUT',
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  DEPENDENCY_FAILURE = 'DEPENDENCY_FAILURE',