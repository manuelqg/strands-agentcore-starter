# Agentic Mindset Week 1

A starter repository with rules and configuration files to help you create your first Strands and AgentCore agents using Amazon Q or Kiro.

## What's Included

- **Rules & Steering Files**: Guidelines for both Amazon Q and Kiro to assist in agent development
- **MCP Configuration**: Optimal MCP server setup for Strands and AgentCore development

## Prerequirements
- Python 3.10+ installed.
- AWS Credentials with permissions of Amazon Bedrock and Amazon AgentCore
- UV installed. Follow instruction [here](https://docs.astral.sh/uv/getting-started/installation/) to install UV in your machine
- Kiro or Amazon Q CLI installed
  
## Quick Start

- Clone this repository `git clone https://github.com/manuelqg/strands-agentcore-starter`
- Open it in Kiro or Amazon Q Developer CLI
- Configure MCP servers. Use `mcp.json` to copy and paste into Kiro. Use `mcp.md` to install in Amazon Q Developer CLI via script.
- Configure an `.env` file with AWS Credentials
- Send your first prompt:

```
Create a Strands Agent in cdk_agent.py that connects to the AWS CDK MCP server.
Use the MCP server: awslabs.cdk-mcp-server@latest. Use .env file to authenticate with AWS.
Test the agent to make sure it works. Use steering files for guidance
```

- Follow up prompt:

```
Deploy a a simple weather agent to AgentCore using strands framework. Use credential from .env file to authenticate. 
Name the file agentcore_weather_agent. Test the agent. 
Deploy to us-east-1. Use agentcore and strands mcp for guidance
”
```

## Resources

- [Strands Agents Documentation](https://strandsagents.com/latest/)
- [AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [MCP Specification](https://github.com/modelcontextprotocol/specification)
