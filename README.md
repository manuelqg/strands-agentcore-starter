# Strands AgentCore Starter

A starter repository with rules and configuration files to help you create your first Strands and AgentCore agents using Amazon Q or Kiro.

## What's Included

- **Rules & Steering Files**: Guidelines for both Amazon Q and Kiro to assist in agent development
- **MCP Configuration**: Optimal MCP server setup for Strands and AgentCore development
- **Best Practices**: Development patterns and troubleshooting guides

## Quick Start

### Prerequisites
- AWS CLI configured with appropriate permissions
- Python 3.10 or newer
- Virtual environment setup

### Setup
```bash
# Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install required packages
pip install "bedrock-agentcore-starter-toolkit>=0.1.21" strands-agents boto3
```

### Create Your First Agent
1. Use the rules in `.amazonq/rules/` to guide development
2. Configure MCP servers using the provided `mcp.json`
3. Follow the AgentCore development patterns in the documentation

## Files Structure

- `.amazonq/rules/agentcore.md` - AgentCore development guidelines
- `.amazonq/rules/strands-python.md` - Strands Agents best practices
- `mcp.json` - MCP server configuration

## Getting Help

Use Amazon Q or Kiro with the included rules to get assistance with:
- Agent development patterns
- MCP integration
- Deployment and configuration
- Troubleshooting common issues

## Resources

- [Strands Agents Documentation](https://strandsagents.com/latest/)
- [AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [MCP Specification](https://github.com/modelcontextprotocol/specification)
