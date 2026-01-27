# Pulumi Agent Skills

A collection of [Agent Skills](https://agentskills.io) for infrastructure as code workflows with Pulumi. These skills teach AI coding assistants how to help with infrastructure migrations, secret management, and code translation.

## What are Agent Skills?

Agent Skills are reusable knowledge packages that teach AI coding assistants domain-specific workflows. They follow the [agentskills.io](https://agentskills.io) open standard and work with:

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [GitHub Copilot](https://docs.github.com/en/copilot)
- [Cursor](https://cursor.sh)
- [VS Code](https://code.visualstudio.com/docs/copilot)

## Available Skills

| Skill | Description |
|-------|-------------|
| **pulumi-terraform-to-pulumi** | Convert Terraform HCL to Pulumi |
| **pulumi-cdk-to-pulumi** | Migrate AWS CDK applications to Pulumi |
| **pulumi-cdk-convert** | Convert CDK Cloud Assemblies to Pulumi YAML |
| **pulumi-cdk-importer** | Import CDK-managed infrastructure into Pulumi state |
| **pulumi-arm-to-pulumi** | Convert Azure ARM/Bicep templates to Pulumi |
| **pulumi-arm-import** | Import existing Azure resources into Pulumi |
| **pulumi-cloudformation-id-lookup** | Look up Pulumi import ID formats for AWS resources |
| **pulumi-esc** | Manage Pulumi ESC (Environments, Secrets, Configuration) |
| **pulumi-best-practices** | Pulumi development patterns and best practices |
| **pulumi-automation-api** | Programmatic orchestration of Pulumi operations |

## Installation

Install using the add-skill CLI:

```bash
npx add-skill pulumi/agent-skills
```

Or clone manually to the appropriate skills directory for your tool:

| Tool | Personal | Project |
| --- | --- | --- |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| GitHub Copilot | - | `.github/skills/` |
| OpenAI Codex | `~/.codex/skills/` | `.codex/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.gemini/skills/` |

```bash
git clone https://github.com/pulumi/agent-skills.git <skills-directory>/pulumi
```

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
Set up AWS OIDC credentials using Pulumi ESC"
```

The assistant will use the `pulumi-esc` skill to help configure dynamic credentials.

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on:

- Writing new skills
- Improving existing skills
- Reporting issues

## License

Apache 2.0 - See [LICENSE](LICENSE) for details.

## Resources

- [Pulumi Documentation](https://www.pulumi.com/docs/)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Pulumi ESC Documentation](https://www.pulumi.com/docs/esc/)
