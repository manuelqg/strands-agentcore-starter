---
inclusion: fileMatch
fileMatchPattern: ['**/*.py', '**/requirements*.txt', '**/*.md']
---
# Strands Agents and Python Development Guidelines

## About Strands Agents

Strands Agents is a simple yet powerful SDK for building AI agents in Python. It takes a model-driven approach and scales from simple conversational assistants to complex autonomous workflows.

### Key Features
- **Lightweight & Flexible**: Simple agent loop that just works and is fully customizable
- **Model Agnostic**: Support for Amazon Bedrock, Anthropic, Llama, Ollama, and custom providers
- **Advanced Capabilities**: Multi-agent systems, autonomous agents, and streaming support
- **Built-in MCP**: Native support for Model Context Protocol (MCP) servers

## Python Best Practices for Strands Agents

## Always use virtual environments. 
- Create a virtual a environment under a .venv directory
- Always activate virtual environment before installing dependencies
- Always activate virtual environment before running any agent


### Code Structure
- Use clear, descriptive variable names for agents and tools
- Implement proper error handling with try/catch blocks
- Use type hints where appropriate for better code clarity
- Follow PEP 8 style guidelines

### Agent Development Patterns
- Keep system prompts clear and specific
- Use the `@tool` decorator for creating agent tools
- Implement proper resource management with context managers (`with` statements)
- Structure multi-agent systems with clear separation of concerns

### MCP Integration
- Use `MCPClient` for connecting to MCP servers
- Implement proper connection handling with context managers
- Handle MCP server errors gracefully
- Use `uvx` for running MCP servers when possible
- **MCPAgentTool Attributes**: `MCPAgentTool` objects don't expose name/description attributes directly. Simply pass the tools list to the Agent without trying to inspect individual tool properties. The agent will handle tool discovery internally.

### Example Code Structure
from strands import Agent
from strands.models import BedrockModel
from strands.tools.mcp import MCPClient
from mcp import StdioServerParameters, stdio_client

# Initialize model
model = BedrockModel(
    model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
    temperature=0.7,
)

# Create MCP client
mcp_client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="uvx", 
            args=["server-name@latest"]
        )
    )
)

# Use context manager for proper resource handling
with mcp_client:
    tools = mcp_client.list_tools_sync()
    
    # Don't try to inspect tool.name or tool.definition - just pass tools to Agent
    print(f"Connected with {len(tools)} tools")
    
    agent = Agent(tools=tools, model=model, system_prompt="Your system prompt here")
    response = agent("Your query here")
    print(response)


### Testing and Debugging
- Test agents with simple queries first
- Use print statements or logging for debugging agent interactions
- Validate MCP server connections before running agents
- Handle network timeouts and connection errors gracefully
- **MCPAgentTool Inspection**: Don't try to access `tool.name`, `tool.definition`, or other attributes on `MCPAgentTool` objects. They are opaque wrappers. Just count them with `len(tools)` and pass them to the Agent.

## Resources
- [Strands Agents Documentation](https://strandsagents.com/latest/)
- [MCP Specification](https://github.com/modelcontextprotocol/specification)
- [AWS MCP Servers](https://github.com/awslabs/mcp)
