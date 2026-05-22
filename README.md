Overall System Architecture: Custom Metadata IntegrationProject: Custom Metadata Analytics (x-metadata-* Headers)Status: High-Level DesignComponents Involved: api-graph, api-llm, api-llm-track, api-llmtrack-ui, PostgreSQL1. System ObjectiveThe system allows clients to attach arbitrary, custom metadata tags to their LLM and Agent API requests using HTTP headers prefixed with x-metadata- (e.g., x-metadata-tenant-id: acme).The architecture guarantees that these tags are extracted at the network edge, securely propagated through internal microservices, stored efficiently without altering database schemas, and ultimately made available in the UI for dynamic filtering and grouping (segmentation).2. High-Level Architecture Diagramflowchart TD
    Client([Client / Customer])
    
    subgraph Edge Services
        Graph[api-graph <br/> Agent Orchestrator]
        LLM[api-llm <br/> LLM Gateway]
    end
    
    subgraph Telemetry & Storage
        Track[api-llm-track <br/> Telemetry Service]
        DB[(PostgreSQL <br/> trace_new table)]
    end
    
    subgraph User Interface
        UI[api-llmtrack-ui <br/> Analytics Dashboard]
    end
    
    External[External LLM Providers <br/> OpenAI, Anthropic, etc.]

    Client -- "1. HTTP Request + \nx-metadata-* headers" --> Graph
    Client -- "1. Direct HTTP Request + \nx-metadata-* headers" --> LLM
    
    Graph -- "2. Forwards metadata \nas headers" --> LLM
    Graph -- "2. Agent Lifecycle Events \n(body payload)" --> Track
    
    LLM -- "3. Strips metadata \n(Security Scrubbing)" --> External
    LLM -- "4. Sends Trace Data \n+ metadata payload" --> Track
    
    Track -- "5. Validates & Inserts \nJSONB" --> DB
    
    UI -- "6. Fetches Filter Keys \n& Segmented Data" --> Track
3. Component ResponsibilitiesA. Network Edge (api-graph & api-llm)These services act as the entry points for external traffic. Their responsibilities include:Extraction & Normalization: Intercepting any header starting with x-metadata-, converting the key to lowercase, and stripping the prefix.Edge Validation: Enforcing strict limits to prevent abuse (Max 10 distinct headers, max 64-character keys, max 256-character values). Invalid requests are rejected immediately with a 400 Bad Request.Context Propagation: Passing the metadata downstream. api-graph forwards it as HTTP headers to api-llm. Both services append it to the OpenTelemetry span attributes for debugging.Security Scrubbing (Crucial): api-llm acts as the security boundary. It must strip all x-metadata-* headers before forwarding prompts to external LLM providers (e.g., OpenAI, Azure) to prevent data leakage.B. Telemetry Backend (api-llm-track)This service acts as the source of truth for observability data.Defense-in-Depth Validation: Re-validates the incoming metadata payload before database interaction.Persistence: Writes the metadata to the trace_new table.Query Translation: Translates frontend query parameters (e.g., ?metadata[tenant-id]=acme) into optimized PostgreSQL queries.C. Database (PostgreSQL)Storage Structure: Uses a single, nullable JSONB column named metadata on the trace_new table. This provides flexible schema-less storage (allowing customers to invent tags on the fly) while maintaining relational integrity.Indexing Strategy: Uses a Generalized Inverted Index (GIN) with the jsonb_path_ops operator. This allows lightning-fast containment queries (e.g., checking if a JSON object contains {"tenant-id": "acme"}) over hundreds of millions of rows.D. User Interface (api-llmtrack-ui)Dynamic Discovery: Queries the backend for all distinct metadata keys used within a specific time window to dynamically populate dashboard filter dropdowns.Analytics Segmentation: Appends user-selected metadata filters to existing analytics API calls, enabling cost, latency, and token-usage breakdowns by custom customer dimensions.4. Key Architectural GuaranteesZero External Leakage: Metadata is strictly internal telemetry data. It never crosses the boundary to external AI model providers.Backward Compatibility: All existing traces without metadata default to NULL in the database. Older API clients sending requests without these headers experience zero behavioral changes.Tenant Isolation: Metadata filtering is strictly AND-combined with existing client_id and project_id constraints. A user can never query or discover metadata tags belonging to a different tenant.Performance Safety: The use of jsonb_path_ops ensures that arbitrary tag creation by users does not degrade database read speeds or require continuous schema migrations by the platform team.
