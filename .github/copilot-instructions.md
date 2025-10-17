# AI Agent Development Instructions

This Azure AI Foundry agent application follows specific patterns and workflows. Follow these guidelines for effective contribution.

## Architecture Overview

### Core Components
- **Azure AI Project**: Central hub using Azure AI Agent service with file search/Azure AI Search for knowledge retrieval
- **FastAPI Backend** (`src/api/`): Handles agent interactions, streaming responses, and automatic evaluations
- **Container Apps Deployment**: Scalable hosting on Azure with integrated monitoring and tracing
- **Bicep Infrastructure** (`infra/`): Complete Azure resource provisioning with security roles and monitoring

### Agent Lifecycle
```
Deployment → Agent Creation (gunicorn.conf.py) → Request Handling (routes.py) → Streaming Response → Auto-evaluation
```

## Key Development Patterns

### Agent Configuration
- **Agent creation**: Modify `src/gunicorn.conf.py` around line 175 for instructions/tools
- **Existing agents**: Use Azure AI Foundry UI instead of code changes to preserve conversation history
- **Tool selection**: Agent automatically chooses between `AzureAISearchTool` or `FileSearchTool` based on environment
- **Instructions**: Default patterns are "Use AI Search always" or "Use File Search always"

### Environment Variables Pattern
Critical variables set via Bicep deployment in `infra/api.bicep`:
```bash
AZURE_EXISTING_AIPROJECT_ENDPOINT    # Project endpoint for client connection
AZURE_EXISTING_AGENT_ID              # Agent ID (fallback to name lookup)
AZURE_AI_AGENT_DEPLOYMENT_NAME       # Model deployment (default: gpt-4o-mini)
ENABLE_AZURE_MONITOR_TRACING         # Controls OpenTelemetry tracing
```

### Request Flow & Streaming
- **SSE Streaming**: All agent responses use Server-Sent Events via `StreamingResponse`
- **Event types**: `message`, `completed_message`, `thread_run`, `stream_end`, `error`
- **Thread management**: Cookies persist thread_id and agent_id across requests
- **Auto-evaluation**: Triggered on thread run completion when tracing enabled

### File Uploads & Knowledge
- **File location**: Place knowledge files in `src/files/` - automatically uploaded during agent creation
- **Search integration**: Azure AI Search uses pre-built index from `src/data/embeddings.csv`
- **Tool selection logic**: Checks for search connection → AI Search, else falls back to file search with vector store

## Development Workflows

### Local Development
```bash
# After azd up deployment
cd src && python -m pip install -r requirements.txt
python -m uvicorn "api.main:create_app" --factory --reload
```

### Frontend Customization
- **Key component**: `src/frontend/src/components/agents/AgentPreview.tsx`
- **Build process**: `cd src/frontend && pnpm build` → outputs to `../api/static/react`
- **Backend communication**: All SSE handling and message flow logic in AgentPreview.tsx

### Evaluation & Testing
- **Local evaluation**: Run `python evals/evaluate.py` (requires agent deployment)
- **Automatic evaluation**: Enabled when Application Insights configured - runs after each thread completion
- **Test queries**: Modify `evals/eval-queries.json` for custom evaluation scenarios

## Infrastructure Conventions

### Bicep Development (follow `.github/instructions/bicep-code-best-practices.instructions.md`)
- **Naming**: lowerCamelCase for all resources, avoid 'name' suffixes
- **Parameters**: Use @description decorators, set safe defaults for testing
- **References**: Use symbolic names (`resource.property`) over functions
- **Security**: Never expose secrets in outputs, use managed identity roles

### Deployment Customization
Set environment variables before `azd up`:
```bash
azd env set AZURE_AI_AGENT_MODEL_NAME gpt-4o           # Change model
azd env set AZURE_AI_AGENT_DEPLOYMENT_CAPACITY 100     # Increase quota  
azd env set ENABLE_AZURE_MONITOR_TRACING true          # Enable tracing
azd env set USE_APPLICATION_INSIGHTS true              # Enable monitoring
```

### Security & Authentication
- **Managed Identity**: All Azure resource access uses DefaultAzureCredential
- **Role assignments**: Automatic RBAC setup in `infra/main.bicep`
- **Optional Basic Auth**: Set `WEB_APP_USERNAME`/`WEB_APP_PASSWORD` for web access
- **API version**: Uses `2025-05-15-preview` for evaluation features

## Testing & Monitoring

### Error Handling
- **Global exception handler**: All unhandled exceptions logged in routes.py
- **Agent failures**: Check `run.last_error` in thread run responses
- **Deployment issues**: Use `azd show` for resource group links, check Container App logs

### Observability
- **Tracing**: Enable via `ENABLE_AZURE_MONITOR_TRACING=true` → View in AI Foundry "Tracing" tab
- **Logging**: Uses structured logging via `logging_config.py` 
- **Evaluation metrics**: Continuous monitoring with built-in evaluators (Relevance, TaskAdherence, ToolCallAccuracy)

### Sample Questions
Test deployments using queries from `docs/sample_questions.md` to validate agent performance.

## Project-Specific Commands

```bash
# Test agent configuration
cd src && python gunicorn.conf.py

# Run evaluations  
python evals/evaluate.py

# Deploy with custom settings
azd env set AZURE_AI_AGENT_MODEL_NAME gpt-4o && azd up

# Clean up resources
azd down
```