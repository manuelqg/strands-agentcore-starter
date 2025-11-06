---
inclusion: fileMatch
fileMatchPattern: ['**/*.py', '**/requirements*.txt', '**/*.md', '**/.bedrock_agentcore.yaml']
---
# Amazon Bedrock AgentCore Development Guidelines

## About Amazon Bedrock AgentCore

Amazon Bedrock AgentCore is a fully managed platform for building and deploying production-ready AI agents with runtime hosting, memory, secure code execution, and observability built-in.

### Key Features
- **AgentCore Runtime**: Serverless deployment and scaling with managed container orchestration
- **AgentCore Memory**: Persistent knowledge with short-term (STM) and long-term memory (LTM) extraction
- **Code Interpreter**: Secure Python code execution in isolated sandboxes
- **Observability**: Real-time monitoring with AWS X-Ray tracing and CloudWatch integration
- **Gateway & Identity**: Transform APIs into agent tools with secure authentication

## Prerequisites and Setup

### Required Permissions
- AWS root users or users with AdministratorAccess can skip permission setup
- Others need: starter toolkit policy + AmazonBedrockAgentCoreFullAccess managed policy

### Required Tools
- AWS CLI version 2.0 or later (configured with `aws configure`)
- Python 3.10 or newer
- Amazon Bedrock model access to Claude 3.7 Sonnet (or your preferred model)

### AWS Region Consistency
Always use the SAME AWS Region for:
- Default Region in `aws configure`
- Amazon Bedrock model access
- All AgentCore resources

## Installation

### Always use virtual environments
```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install required packages (version 0.1.21 or later)
pip install "bedrock-agentcore-starter-toolkit>=0.1.21" strands-agents boto3
```

## Model Selection

### Always use Claude Haiku for AgentCore agents

Always choose `us.anthropic.claude-haiku-4-5-20251001-v1:0` as the default model for AgentCore agents:

```python
from strands.models import BedrockModel

# ✅ CORRECT - Use Claude Haiku
bedrock_model = BedrockModel(
    model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
    temperature=0.7,
)
```

**Why Claude Haiku?**
- **Cost-effective**: Lowest cost per token among Claude models
- **Fast inference**: Quick response times for agent operations
- **Sufficient capability**: Handles complex reasoning and tool use
- **Optimized for AgentCore**: Designed for production agent workloads
- **Reliable**: Proven performance in production environments

**Model Comparison:**

| Model | Cost | Speed | Reasoning | Best For |
|-------|------|-------|-----------|----------|
| Claude Haiku | ✅ Lowest | ✅ Fastest | Good | AgentCore agents, real-time responses |
| Claude Sonnet | Medium | Medium | Excellent | Complex analysis, research |
| Claude Opus | Highest | Slowest | Best | Complex reasoning, edge cases |

**When to use other models:**
- Use **Sonnet** only if Haiku cannot handle your specific use case
- Use **Opus** only for complex multi-step reasoning that Sonnet cannot handle
- Always start with Haiku and upgrade only if needed

## Agent Development Patterns

### Basic Agent Structure
```python
"""
Strands Agent sample with AgentCore
"""
import os
from strands import Agent, tool
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig, RetrievalConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager
from bedrock_agentcore.tools.code_interpreter_client import CodeInterpreter
from bedrock_agentcore.runtime import BedrockAgentCoreApp

# Bypass tool consent for automated execution
os.environ["BYPASS_TOOL_CONSENT"] = "true"

app = BedrockAgentCoreApp()

MEMORY_ID = os.getenv("BEDROCK_AGENTCORE_MEMORY_ID")
REGION = os.getenv("AWS_REGION")
MODEL_ID = "us.anthropic.claude-haiku-4-5-20251001-v1:0"

@app.entrypoint
def invoke(payload, context):
    """
    Handler for agent invocation
    
    Args:
        payload: Input payload containing the user prompt
        context: AgentCore context with session information
        
    Returns:
        Dictionary with the agent's response
    """
    # Extract user message from payload
    user_message = payload.get(
        "prompt", 
        "No prompt found in input. Please provide a JSON payload with a 'prompt' key."
    )
    
    # Create agent
    agent = Agent(
        model=MODEL_ID,
        system_prompt="You are a helpful assistant.",
        tools=[your_tools]
    )
    
    # Get agent response
    result = agent(user_message)
    
    # Log context for debugging
    # ⚠️ IMPORTANT: RequestContext only has 'session_id' attribute
    # Do NOT try to access 'request_id' - it doesn't exist!
    print(f"Session ID: {context.session_id}")
    print(f"Result: {result}")
    
    # Return response
    return {"result": result.message}

if __name__ == "__main__":
    app.run()
```

### Memory Configuration
```python
# Configure memory with retrieval settings
memory_config = AgentCoreMemoryConfig(
    memory_id=MEMORY_ID,
    session_id=session_id,
    actor_id=actor_id,
    retrieval_config={
        f"/users/{actor_id}/facts": RetrievalConfig(top_k=3, relevance_score=0.5),
        f"/users/{actor_id}/preferences": RetrievalConfig(top_k=3, relevance_score=0.5)
    }
)

# Create agent with memory
agent = Agent(
    model=MODEL_ID,
    session_manager=AgentCoreMemorySessionManager(memory_config, REGION),
    system_prompt="You are a helpful assistant. Use tools when appropriate.",
    tools=[your_tools]
)
```

### MCP Integration with AgentCore

When integrating MCP (Model Context Protocol) servers with AgentCore, you can connect to multiple MCP servers and aggregate their tools:

```python
import json
import logging
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from mcp import StdioServerParameters, stdio_client
from strands import Agent
from strands.models import BedrockModel
from strands.tools.mcp import MCPClient

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize the AgentCore App
app = BedrockAgentCoreApp()

# Initialize multiple MCP clients at module level
aws_docs_client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="uvx",
            args=["awslabs.aws-documentation-mcp-server@latest"]
        )
    )
)

aws_cdk_client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="uvx",
            args=["awslabs.cdk-mcp-server@latest"]
        )
    )
)

aws_pricing_client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="uvx",
            args=["awslabs.aws-pricing-mcp-server@latest"]
        )
    )
)

# Initialize Bedrock model
# ✅ Always use Claude Haiku for AgentCore agents
bedrock_model = BedrockModel(
    model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
    temperature=0.7,
)

SYSTEM_PROMPT = """You are an expert AWS Solutions Architect with access to:
1. AWS Documentation tools
2. AWS CDK tools
3. AWS Pricing tools

Provide accurate, comprehensive answers using all available tools."""

def _extract_response_text(response) -> str:
    """Extract text from agent response, handling different response types."""
    if isinstance(response, str):
        return response
    
    if hasattr(response, 'message') and isinstance(response.message, dict):
        content = response.message.get("content", [])
        if isinstance(content, list) and len(content) > 0:
            if isinstance(content[0], dict) and "text" in content[0]:
                return content[0]["text"]
            elif isinstance(content[0], str):
                return content[0]
    
    if hasattr(response, 'text'):
        return response.text
    
    return str(response)

@app.entrypoint
def agent_invocation(payload, context):
    """Handler for agent invocation with multiple MCP servers"""
    try:
        if isinstance(payload, str):
            payload = json.loads(payload)
        
        user_message = payload.get("prompt")
        if not user_message:
            return {"error": "No prompt provided"}
        
        logger.info(f"Processing: {user_message}")
        logger.info(f"Session ID: {context.session_id}")
        
        # Collect tools from all MCP servers
        all_tools = []
        
        # Load AWS Documentation tools
        try:
            with aws_docs_client:
                docs_tools = aws_docs_client.list_tools_sync()
                all_tools.extend(docs_tools)
                logger.info(f"Loaded {len(docs_tools)} documentation tools")
        except Exception as e:
            logger.warning(f"Failed to load docs tools: {e}")
        
        # Load AWS CDK tools
        try:
            with aws_cdk_client:
                cdk_tools = aws_cdk_client.list_tools_sync()
                all_tools.extend(cdk_tools)
                logger.info(f"Loaded {len(cdk_tools)} CDK tools")
        except Exception as e:
            logger.warning(f"Failed to load CDK tools: {e}")
        
        # Load AWS Pricing tools
        try:
            with aws_pricing_client:
                pricing_tools = aws_pricing_client.list_tools_sync()
                all_tools.extend(pricing_tools)
                logger.info(f"Loaded {len(pricing_tools)} pricing tools")
        except Exception as e:
            logger.warning(f"Failed to load pricing tools: {e}")
        
        logger.info(f"Total tools available: {len(all_tools)}")
        
        # Create agent with all tools
        agent = Agent(
            tools=all_tools,
            model=bedrock_model,
            system_prompt=SYSTEM_PROMPT
        )
        
        # Get response from agent
        logger.info("Invoking agent...")
        response = agent(user_message)
        
        # Extract and return response
        response_text = _extract_response_text(response)
        logger.info(f"Response: {len(response_text)} characters")
        
        return {"response": response_text}
        
    except Exception as e:
        logger.error(f"Error: {e}", exc_info=True)
        return {"error": str(e)}

app.run()
```

**Key Points for Multi-MCP Integration:**
- Initialize MCP clients at module level for reuse across invocations
- Use context managers (`with mcp_client:`) for proper connection handling
- Aggregate tools from multiple MCP servers into a single list
- Handle exceptions gracefully when MCP servers are unavailable
- Use logging to track tool loading and agent invocation
- Extract response text properly from different response types
- Set `BYPASS_TOOL_CONSENT=true` for automated tool execution
- MCP servers run via `uvx` command (requires `uv` package manager)

**Available AWS MCP Servers:**
- `awslabs.aws-documentation-mcp-server@latest` - AWS documentation search and retrieval
- `awslabs.cdk-mcp-server@latest` - AWS CDK guidance and best practices
- `awslabs.aws-pricing-mcp-server@latest` - AWS pricing information
- `awslabs.aws-cdk-nag-mcp-server@latest` - CDK Nag security checks

### Code Interpreter Integration
```python
from bedrock_agentcore.tools.code_interpreter_client import CodeInterpreter

ci_sessions = {}

@tool
def calculate(code: str) -> str:
    """Execute Python code for calculations or analysis."""
    session_id = current_session or 'default'
    
    if session_id not in ci_sessions:
        ci_sessions[session_id] = {
            'client': CodeInterpreter(REGION),
            'session_id': None
        }
    
    ci = ci_sessions[session_id]
    if not ci['session_id']:
        ci['session_id'] = ci['client'].start(
            name=f"session_{session_id[:30]}",
            session_timeout_seconds=1800
        )
    
    result = ci['client'].invoke("executeCode", {
        "code": code,
        "language": "python"
    })
    
    for event in result.get("stream", []):
        if stdout := event.get("result", {}).get("structuredContent", {}).get("stdout"):
            return stdout
    return "Executed"
```

## AgentCore CLI Commands

### Configuration
```bash
# Interactive configuration (prompts for all options)
agentcore configure --entrypoint your_agent.py

# Non-interactive configuration (uses defaults)
agentcore configure --entrypoint your_agent.py \
  --name agent_name \
  --requirements-file requirements.txt \
  --disable-memory \
  --non-interactive \
  --region us-east-1

# Configuration with specific options
agentcore configure --entrypoint your_agent.py \
  --name my_agent \
  --execution-role arn:aws:iam::123456789012:role/MyRole \
  --ecr auto \
  --region us-east-1

# Interactive prompts will ask for:
# 1. Execution Role (press Enter to auto-create)
# 2. ECR Repository (press Enter to auto-create)
# 3. Requirements File (confirm or specify path)
# 4. OAuth Configuration (type 'no' for basic setup)
# 5. Request Header Allowlist (type 'no' for basic setup)
# 6. Memory Configuration (type 'yes' to enable LTM extraction)
```

### Deployment
```bash
# Deploy agent to AgentCore Runtime (default: CodeBuild)
agentcore launch

# Deploy specific agent
agentcore launch --agent agent_name

# Auto-update existing agent instead of failing
agentcore launch --auto-update-on-conflict

# Local development mode (requires Docker/Finch/Podman)
agentcore launch --local

# Build locally, deploy to cloud
agentcore launch --local-build

# Deploy with environment variables
agentcore launch --env API_KEY=abc123 --env DEBUG=true

# Deployment performs:
# - Memory resource provisioning (if enabled)
# - Docker container build with dependencies
# - ECR repository push
# - AgentCore Runtime deployment with X-Ray tracing
# - CloudWatch Transaction Search configuration
# - Endpoint activation with trace collection
```

### Monitoring
```bash
# Check deployment status (default agent)
agentcore status

# Check specific agent status
agentcore status --agent agent_name

# Verbose output with full JSON
agentcore status --verbose

# View runtime logs in real-time
aws logs tail /aws/bedrock-agentcore/runtimes/AGENT_ID-DEFAULT \
  --log-stream-name-prefix "YYYY/MM/DD/[runtime-logs]" --follow

# View recent logs (last hour)
aws logs tail /aws/bedrock-agentcore/runtimes/AGENT_ID-DEFAULT \
  --log-stream-name-prefix "YYYY/MM/DD/[runtime-logs]" --since 1h
```

### Testing
```bash
# Invoke agent (default agent)
agentcore invoke '{"prompt": "Your prompt here"}'

# Invoke specific agent
agentcore invoke '{"prompt": "Your prompt"}' --agent agent_name

# Invoke with specific session ID (for conversation continuity)
SESSION_ID=$(python -c "import uuid; print(uuid.uuid4())")
agentcore invoke '{"prompt": "Your prompt"}' --session-id $SESSION_ID

# Invoke with OAuth authentication
agentcore invoke '{"prompt": "Secure request"}' --bearer-token YOUR_TOKEN

# Invoke with custom headers
agentcore invoke '{"prompt": "Test"}' --headers "Actor-Id:user123,Trace-Id:abc"

# Invoke local agent (for testing before deployment)
agentcore invoke '{"prompt": "Test locally"}' --local

# Stop active session to free resources
agentcore stop-session --session-id $SESSION_ID
```

### Cleanup
```bash
# Preview what will be destroyed (dry run)
agentcore destroy --dry-run

# Destroy with confirmation prompt
agentcore destroy

# Destroy specific agent without confirmation
agentcore destroy --agent agent_name --force

# Destroy and delete ECR repository
agentcore destroy --agent agent_name --delete-ecr-repo

# Removes:
# - AgentCore Runtime endpoint and agent
# - AgentCore Memory resources (STM and LTM)
# - Amazon ECR images
# - CodeBuild project
# - IAM roles (if auto-created and not used by other agents)
# - Agent deployment configuration
```

## Memory Best Practices

### Short-Term Memory (STM)
- Always enabled by default
- Persists within a single session
- Immediate context retention
- No extraction delay

### Long-Term Memory (LTM)
- Enable during `agentcore configure` by typing 'yes'
- Extracts facts across sessions (takes 10-30 seconds)
- Wait 15-30 seconds after storing facts before starting new session
- Memory provisioning takes 2-5 minutes initially
- Check status with `agentcore status` before testing

### Session Management
- Session IDs must be 33+ characters
- Use UUIDs for unique sessions: `python -c "import uuid; print(uuid.uuid4())"`
- Actor IDs help organize user-specific memory paths

## Observability

### CloudWatch Dashboard
- Access GenAI Observability dashboard from `agentcore status` output
- View end-to-end request traces
- Monitor memory retrieval operations
- Track code interpreter executions
- Analyze latency breakdown by component

### X-Ray Tracing
- Automatically configured during deployment
- Provides distributed tracing across all services
- View traces: AWS Console → CloudWatch → Service Map or X-Ray → Traces
- Wait 30-60 seconds for traces to appear

### Logs
- Runtime logs: `/aws/bedrock-agentcore/runtimes/AGENT_ID-DEFAULT`
- Memory logs: `/aws/vendedlogs/bedrock-agentcore/memory/APPLICATION_LOGS/memory-id`
- Use `--since 1h` flag for recent logs

## Common Errors and Solutions

### RequestContext AttributeError
**Error**: `AttributeError: 'RequestContext' object has no attribute 'request_id'`

**Cause**: The AgentCore `RequestContext` object only has `session_id` attribute, not `request_id`.

**Solution**:
```python
# ❌ WRONG - This will cause AttributeError
print(f"Request ID: {context.request_id}")

# ✅ CORRECT - Only session_id exists
print(f"Session ID: {context.session_id}")
```

**Available RequestContext Attributes:**
- `context.session_id` - The session identifier (string)
- That's it! No other attributes are available on the context object.

### Agent Naming Rules
**Error**: `Invalid agent name. Must start with a letter, contain only letters/numbers/underscores`

**Solution**: Agent names must:
- Start with a letter
- Contain only letters, numbers, and underscores (no hyphens!)
- Be 1-48 characters long

```bash
# ❌ WRONG
agentcore configure --name my-agent  # Hyphens not allowed

# ✅ CORRECT
agentcore configure --name my_agent  # Underscores are fine
agentcore configure --name myagent   # No separators also works
```

## Troubleshooting

### Memory configuration not appearing
```bash
# Verify toolkit version (must be >= 0.1.21)
pip show bedrock-agentcore-starter-toolkit

# If outdated, reinstall in venv
deactivate
source .venv/bin/activate
pip install --force-reinstall --no-cache-dir "bedrock-agentcore-starter-toolkit>=0.1.21"

# Verify correct path
which agentcore  # Should show .venv/bin/agentcore
```

### Memory status not active
- Run `agentcore status` to check memory status
- If "provisioning", wait 2-3 minutes
- Retry after status shows "STM+LTM (3 strategies)"

### Cross-session memory not working
- Verify LTM is active (not "provisioning")
- Wait 15-30 seconds after storing facts for extraction
- Check extraction logs for completion

### Wrong AWS Region
```bash
# Clean up resources in incorrect region
agentcore destroy

# Verify/reconfigure AWS CLI region
aws configure get region
aws configure set region your-desired-region

# Verify Bedrock model access in target region
# Then reconfigure and redeploy
```

## Code Structure Best Practices

- Use environment variables for MEMORY_ID and REGION
- Implement proper error handling for memory provisioning states
- Manage Code Interpreter sessions per user/session
- Use context managers for resource cleanup
- Follow PEP 8 style guidelines
- Add type hints for better code clarity
- Keep system prompts clear and specific

## Resources
- [AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [Strands Agents SDK](https://strandsagents.com/latest/documentation/docs/)
- [Gateway Quickstart](https://github.com/aws/bedrock-agentcore-starter-toolkit/blob/main/documentation/docs/user-guide/gateway/quickstart.md)
- [Identity Quickstart](https://github.com/aws/bedrock-agentcore-starter-toolkit/blob/main/documentation/docs/user-guide/identity/quickstart.md)
