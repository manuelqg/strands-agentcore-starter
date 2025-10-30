# Agentic Mindset Week 1

A starter repository with rules and configuration files to help you create your first Strands and AgentCore agents using Amazon Q or Kiro.

## What's Included

- **Rules & Steering Files**: Guidelines for both Amazon Q and Kiro to assist in agent development
- **MCP Configuration**: Optimal MCP server setup for Strands and AgentCore development

## Prerequirements

- AWS Credentials with permissions of Amazon Bedrock and Amazon AgentCore
- Kiro or Amazon Q CLI installed

## Quick Start

- Clone this repository `git clone https://github.com/manuelqg/strands-agentcore-starter`
- Open it in Kiro or Amazon Q Developer CLI
- Configure an `.env` file with AWS Credentials
- Send your first prompt:

```
Create a Strands Agent in cdk_agent.py that connects to the AWS CDK MCP server.
Use the MCP server: awslabs.cdk-mcp-server@latest. Use .env file to authenticate with AWS.
Test the agent to make sure it works
```

- Follow up prompt:

```
Now deploy my cdk agent to AgentCore. Name the file agentcore_cdk_agent and name my agent cdk_agent. Test the agent.”
```

## Resources

- [Strands Agents Documentation](https://strandsagents.com/latest/)
- [AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [MCP Specification](https://github.com/modelcontextprotocol/specification)
