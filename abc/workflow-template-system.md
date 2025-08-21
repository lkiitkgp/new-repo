# Workflow Template System

## Overview

The workflow template system provides generic, reusable patterns for common business processes. Templates enable rapid workflow creation while maintaining consistency and best practices.

## Template Architecture

### Template Definition Schema

```typescript
interface WorkflowTemplate {
  id: string;
  name: string;
  description: string;
  category: TemplateCategory;
  version: string;
  
  // Template structure
  templateTasks: TemplateTask[];
  templateDependencies: TemplateDependency[];
  
  // Parameterization
  parameters: TemplateParameter[];
  defaultValues: Record<string, any>;
  validationRules: ValidationRule[];
  
  // Metadata
  complexity: ComplexityLevel;
  estimatedDuration: number;
  requiredAgentTypes: string[];
  usageCount: number;
  successRate: number;
  
  // Template configuration
  customizable: boolean;
  approvalRequired: boolean;
  tags: string[];
}

interface TemplateTask {
  id: string;
  name: string;
  description: string;
  type: TaskType;
  
  // Parameterization
  parameterizedFields: ParameterizedField[];
  conditionalInclusion?: TemplateCondition;
  
  // Agent requirements
  agentType: string;
  requiredCapabilities: string[];
  
  // Configuration
  timeout: number | ParameterReference;
  retryPolicy: RetryPolicy;
  criticalityLevel: CriticalityLevel;
  
  // State operations
  stateReads: TemplateStateOperation[];
  stateWrites: TemplateStateOperation[];
}

interface TemplateParameter {
  name: string;
  type: ParameterType;
  description: string;
  required: boolean;
  defaultValue?: any;
  validationRules: ParameterValidationRule[];
  options?: ParameterOption[];
}

enum ParameterType {
  STRING = 'STRING',
  NUMBER = 'NUMBER',
  BOOLEAN = 'BOOLEAN',
  ENUM = 'ENUM',
  OBJECT = 'OBJECT',
  ARRAY = 'ARRAY'
}

enum TemplateCategory {
  DATA_PROCESSING = 'DATA_PROCESSING',
  CUSTOMER_SERVICE = 'CUSTOMER_SERVICE',
  CONTENT_CREATION = 'CONTENT_CREATION',
  APPROVAL_WORKFLOW = 'APPROVAL_WORKFLOW',
  NOTIFICATION_SYSTEM = 'NOTIFICATION_SYSTEM',
  INTEGRATION_WORKFLOW = 'INTEGRATION_WORKFLOW',
  ANALYSIS_WORKFLOW = 'ANALYSIS_WORKFLOW'
}
```

## Built-in Templates

### 1. Customer Onboarding Template

```typescript
const customerOnboardingTemplate: WorkflowTemplate = {
  id: 'customer-onboarding-v1',
  name: 'Customer Onboarding Process',
  description: 'Standard customer onboarding with verification, setup, and welcome communications',
  category: TemplateCategory.CUSTOMER_SERVICE,
  version: '1.0.0',
  
  templateTasks: [
    {
      id: 'identity-verification',
      name: 'Verify Customer Identity',
      description: 'Verify customer identity using provided documents',
      type: TaskType.VALIDATION,
      agentType: 'VERIFICATION_AGENT',
      requiredCapabilities: ['document_analysis', 'identity_verification'],
      timeout: 300000, // 5 minutes
      criticalityLevel: CriticalityLevel.HIGH,
      parameterizedFields: [
        {
          field: 'verification_method',
          parameter: 'verification_method'
        },
        {
          field: 'required_documents',
          parameter: 'required_documents'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'customer_data'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'verification_result'
        }
      ]
    },
    {
      id: 'account-setup',
      name: 'Set Up Customer Account',
      description: 'Create customer account and configure initial settings',
      type: TaskType.DATA_PROCESSING,
      agentType: 'ACCOUNT_AGENT',
      requiredCapabilities: ['account_creation', 'system_integration'],
      timeout: 180000, // 3 minutes
      criticalityLevel: CriticalityLevel.MEDIUM,
      parameterizedFields: [
        {
          field: 'account_type',
          parameter: 'account_type'
        },
        {
          field: 'initial_settings',
          parameter: 'initial_settings'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'verification_result'
        },
        {
          operation: 'READ_SHARED_DATA',
          key: 'customer_data'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'account_details'
        }
      ]
    },
    {
      id: 'welcome-email',
      name: 'Send Welcome Email',
      description: 'Send personalized welcome email to customer',
      type: TaskType.COMMUNICATION,
      agentType: 'COMMUNICATION_AGENT',
      requiredCapabilities: ['email_composition', 'template_processing'],
      timeout: 120000, // 2 minutes
      criticalityLevel: CriticalityLevel.LOW,
      parameterizedFields: [
        {
          field: 'email_template',
          parameter: 'welcome_email_template'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'account_details'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'communication_log'
        }
      ]
    },
    {
      id: 'followup-scheduling',
      name: 'Schedule Follow-up',
      description: 'Schedule follow-up call or meeting with customer',
      type: TaskType.INTEGRATION,
      agentType: 'SCHEDULING_AGENT',
      requiredCapabilities: ['calendar_integration', 'scheduling'],
      timeout: 180000, // 3 minutes
      criticalityLevel: CriticalityLevel.MEDIUM,
      parameterizedFields: [
        {
          field: 'followup_delay',
          parameter: 'followup_delay_days'
        },
        {
          field: 'meeting_duration',
          parameter: 'meeting_duration'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'account_details'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'followup_scheduled'
        }
      ]
    }
  ],
  
  templateDependencies: [
    {
      taskId: 'account-setup',
      dependsOn: ['identity-verification'],
      dependencyType: DependencyType.SEQUENTIAL
    },
    {
      taskId: 'welcome-email',
      dependsOn: ['account-setup'],
      dependencyType: DependencyType.SEQUENTIAL
    },
    {
      taskId: 'followup-scheduling',
      dependsOn: ['account-setup'],
      dependencyType: DependencyType.SEQUENTIAL
    }
  ],
  
  parameters: [
    {
      name: 'verification_method',
      type: ParameterType.ENUM,
      description: 'Method used for identity verification',
      required: true,
      options: [
        { value: 'document_upload', label: 'Document Upload' },
        { value: 'video_call', label: 'Video Call Verification' },
        { value: 'in_person', label: 'In-Person Verification' }
      ]
    },
    {
      name: 'account_type',
      type: ParameterType.ENUM,
      description: 'Type of account to create',
      required: true,
      options: [
        { value: 'basic', label: 'Basic Account' },
        { value: 'premium', label: 'Premium Account' },
        { value: 'enterprise', label: 'Enterprise Account' }
      ]
    },
    {
      name: 'followup_delay_days',
      type: ParameterType.NUMBER,
      description: 'Number of days to wait before follow-up',
      required: false,
      defaultValue: 7,
      validationRules: [
        { type: 'MIN_VALUE', value: 1 },
        { type: 'MAX_VALUE', value: 30 }
      ]
    }
  ],
  
  complexity: ComplexityLevel.MEDIUM,
  estimatedDuration: 900000, // 15 minutes
  requiredAgentTypes: ['VERIFICATION_AGENT', 'ACCOUNT_AGENT', 'COMMUNICATION_AGENT', 'SCHEDULING_AGENT'],
  customizable: true,
  approvalRequired: false,
  tags: ['customer-service', 'onboarding', 'verification']
};
```

### 2. Data Processing Pipeline Template

```typescript
const dataProcessingTemplate: WorkflowTemplate = {
  id: 'data-processing-pipeline-v1',
  name: 'Data Processing Pipeline',
  description: 'Extract, transform, and load data with validation and quality checks',
  category: TemplateCategory.DATA_PROCESSING,
  version: '1.0.0',
  
  templateTasks: [
    {
      id: 'data-extraction',
      name: 'Extract Data',
      description: 'Extract data from specified source',
      type: TaskType.DATA_PROCESSING,
      agentType: 'EXTRACTION_AGENT',
      requiredCapabilities: ['data_extraction', 'api_integration'],
      timeout: 600000, // 10 minutes
      criticalityLevel: CriticalityLevel.HIGH,
      parameterizedFields: [
        {
          field: 'data_source',
          parameter: 'data_source'
        },
        {
          field: 'extraction_query',
          parameter: 'extraction_query'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'raw_data'
        }
      ]
    },
    {
      id: 'data-validation',
      name: 'Validate Data',
      description: 'Validate extracted data against quality rules',
      type: TaskType.VALIDATION,
      agentType: 'VALIDATION_AGENT',
      requiredCapabilities: ['data_validation', 'quality_assessment'],
      timeout: 300000, // 5 minutes
      criticalityLevel: CriticalityLevel.HIGH,
      parameterizedFields: [
        {
          field: 'validation_rules',
          parameter: 'validation_rules'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'raw_data'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'validation_report'
        }
      ]
    },
    {
      id: 'data-transformation',
      name: 'Transform Data',
      description: 'Transform data according to specified rules',
      type: TaskType.DATA_PROCESSING,
      agentType: 'TRANSFORMATION_AGENT',
      requiredCapabilities: ['data_transformation', 'format_conversion'],
      timeout: 900000, // 15 minutes
      criticalityLevel: CriticalityLevel.MEDIUM,
      conditionalInclusion: {
        parameter: 'skip_transformation',
        condition: 'EQUALS',
        value: false
      },
      parameterizedFields: [
        {
          field: 'transformation_rules',
          parameter: 'transformation_rules'
        },
        {
          field: 'output_format',
          parameter: 'output_format'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'raw_data'
        },
        {
          operation: 'READ_SHARED_DATA',
          key: 'validation_report'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'transformed_data'
        }
      ]
    },
    {
      id: 'data-loading',
      name: 'Load Data',
      description: 'Load processed data to destination',
      type: TaskType.INTEGRATION,
      agentType: 'LOADING_AGENT',
      requiredCapabilities: ['data_loading', 'database_integration'],
      timeout: 600000, // 10 minutes
      criticalityLevel: CriticalityLevel.HIGH,
      parameterizedFields: [
        {
          field: 'destination',
          parameter: 'destination'
        },
        {
          field: 'loading_strategy',
          parameter: 'loading_strategy'
        }
      ],
      stateReads: [
        {
          operation: 'READ_SHARED_DATA',
          key: 'transformed_data'
        }
      ],
      stateWrites: [
        {
          operation: 'WRITE_SHARED_DATA',
          key: 'loading_result'
        }
      ]
    }
  ],
  
  parameters: [
    {
      name: 'data_source',
      type: ParameterType.OBJECT,
      description: 'Data source configuration',
      required: true,
      validationRules: [
        { type: 'REQUIRED_FIELDS', value: ['type', 'connection'] }
      ]
    },
    {
      name: 'skip_transformation',
      type: ParameterType.BOOLEAN,
      description: 'Skip data transformation step',
      required: false,
      defaultValue: false
    },
    {
      name: 'output_format',
      type: ParameterType.ENUM,
      description: 'Output data format',
      required: false,
      defaultValue: 'json',
      options: [
        { value: 'json', label: 'JSON' },
        { value: 'csv', label: 'CSV' },
        { value: 'xml', label: 'XML' },
        { value: 'parquet', label: 'Parquet' }
      ]
    }
  ],
  
  complexity: ComplexityLevel.MEDIUM,
  estimatedDuration: 1800000, // 30 minutes
  requiredAgentTypes: ['EXTRACTION_AGENT', 'VALIDATION_AGENT', 'TRANSFORMATION_AGENT', 'LOADING_AGENT'],
  customizable: true,
  approvalRequired: true, // Data processing often requires approval
  tags: ['data-processing', 'etl', 'pipeline']
};
```

## Template Management System

### Template Engine

```typescript
class TemplateEngine {
  async instantiateTemplate(
    templateId: string,
    parameters: Record<string, any>,
    customizations?: TemplateCustomization[]
  ): Promise<WorkflowDefinition> {
    
    // Load template
    const template = await this.loadTemplate(templateId);
    
    // Validate parameters
    const validation = await this.validateParameters(template, parameters);
    if (!validation.isValid) {
      throw new TemplateValidationError(validation.errors);
    }
    
    // Apply customizations if provided
    const customizedTemplate = customizations ? 
      await this.applyCustomizations(