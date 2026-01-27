# Pulumi Agent Skills

A collection of [Agent Skills](https://agentskills.io) for infrastructure as code workflows with Pulumi. These skills teach AI coding assistants how to help with infrastructure migrations, secret management, and code translation.

## What are Agent Skills?

Agent Skills are reusable knowledge packages that teach AI coding assistants domain-specific workflows. They follow the [agentskills.io](https://agentskills.io) open standard and work with:

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [GitHub Copilot](https://docs.github.com/en/copilot)
- [Cursor](https://cursor.sh)
- [VS Code](https://code.visualstudio.com/docs/copilot)
- [OpenAI Codex](https://openai.com/api/)
- [Gemini CLI](https://geminicli.com)

## Repository Structure

Skills are organized into three plugin groups:

```
pulumi-agent-skills/
├── migration/          # Convert and import from other tools
├── authoring/          # Write quality Pulumi programs
└── configuration/      # Manage secrets and config with ESC
```

## Available Skills

### Migration Plugin

Convert and import infrastructure from other tools to Pulumi:

| Skill | Description |
|-------|-------------|
| [pulumi-terraform-to-pulumi](migration/skills/pulumi-terraform-to-pulumi) | Migrate Terraform projects to Pulumi |
| [pulumi-cdk-to-pulumi](migration/skills/pulumi-cdk-to-pulumi) | Convert AWS CDK applications to Pulumi |
| [pulumi-cdk-convert](migration/skills/pulumi-cdk-convert) | Automated CDK conversion using the cdk2pulumi tool |
| [pulumi-cdk-importer](migration/skills/pulumi-cdk-importer) | Automated import of CDK-managed AWS infrastructure |
| [pulumi-arm-to-pulumi](migration/skills/pulumi-arm-to-pulumi) | Convert Azure ARM templates and Bicep to Pulumi |
| [pulumi-arm-import](migration/skills/pulumi-arm-import) | Import existing Azure resources into Pulumi |
| [pulumi-cloudformation-id-lookup](migration/skills/pulumi-cloudformation-id-lookup) | Look up CloudFormation import identifiers |

### Authoring Plugin

Write quality Pulumi programs, components, and automation:

| Skill | Description |
|-------|-------------|
| [pulumi-best-practices](authoring/skills/pulumi-best-practices) | Best practices for writing reliable Pulumi programs |
| [pulumi-component](authoring/skills/pulumi-component) | Guide for authoring ComponentResource classes |
| [pulumi-automation-api](authoring/skills/pulumi-automation-api) | Best practices for using Pulumi Automation API |

### Configuration Plugin

Manage secrets, configuration, and credentials:

| Skill | Description |
|-------|-------------|
| [esc](configuration/skills/pulumi-esc) | Guidance for working with Pulumi ESC (Environments, Secrets, and Configuration) |

## Installation

### Claude Code Plugin System

```bash
/plugin marketplace add pulumi/agent-skills
/plugin install pulumi-migration     # Install migration skills
/plugin install pulumi-authoring     # Install authoring skills
/plugin install pulumi-configuration # Install configuration skills
```

This also configures the [Pulumi MCP server](https://www.pulumi.com/docs/pulumi-cloud/developer-portals/mcp/), giving your agent access to your Pulumi Cloud organization data.

### Universal (all agents)

```bash
npx skills add pulumi/agent-skills
```

This works with Claude Code, Cursor, Copilot, Codex, Gemini CLI, and other agent tools.

## Usage Examples

### Terraform to Pulumi Migration

Ask your AI assistant:
> "Convert this Terraform configuration to Pulumi TypeScript"

The assistant will use the `pulumi-terraform-to-pulumi` skill to produce idiomatic Pulumi code.

### CDK to Pulumi Migration

Ask your AI assistant:

```text
Help me migrate my CDK application to Pulumi
```

The assistant will use the `pulumi-cdk-to-pulumi` skill to guide you through the complete migration workflow.

### Managing Secrets with ESC

Ask your AI assistant:

```text
Set up AWS OIDC credentials using Pulumi ESC
```

The assistant will use the `esc` skill to help configure dynamic credentials.

### Writing Components

Ask your AI assistant:

```text
Help me create a reusable Pulumi component for a web service
```

The assistant will use the `pulumi-component` skill to guide you through component authoring best practices.

## MCP Server Integration

When you install plugins through Claude Code, the [Pulumi MCP server](https://www.pulumi.com/docs/pulumi-cloud/developer-portals/mcp/) is automatically configured. This provides:

- Access to your Pulumi Cloud organization data
- Cross-stack visibility through Pulumi Insights
- Live infrastructure context
- Registry schema information

The MCP server requires the `PULUMI_ACCESS_TOKEN` environment variable. [Create an access token](https://www.pulumi.com/docs/pulumi-cloud/access-management/access-tokens/) and set it in your shell profile.

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on:

- Writing new skills
- Improving existing skills
- Reporting issues

Also see [AGENTS.md](AGENTS.md) for agent-specific documentation on skill conventions, cross-skill references, and plugin structure.

## License

Apache 2.0 - See [LICENSE](LICENSE) for details.

## Resources

- [Pulumi Documentation](https://www.pulumi.com/docs/)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Pulumi ESC Documentation](https://www.pulumi.com/docs/esc/)
- [Pulumi MCP Server](https://www.pulumi.com/docs/pulumi-cloud/developer-portals/mcp/)
