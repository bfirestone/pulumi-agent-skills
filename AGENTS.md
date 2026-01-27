# Pulumi Agent Skills

This repository contains official Pulumi agent skills for infrastructure as code workflows.

## Repository Structure

Skills are organized into two plugin groups:

### Migration Plugin (`migration/`)

Skills for converting and importing infrastructure from other tools to Pulumi:

- **pulumi-terraform-to-pulumi**: Migrate Terraform projects to Pulumi
- **pulumi-cdk-to-pulumi**: Convert AWS CDK applications to Pulumi
- **pulumi-cdk-convert**: Automated CDK conversion using the cdk2pulumi tool
- **pulumi-cdk-importer**: Automated import of CDK-managed AWS infrastructure
- **pulumi-arm-to-pulumi**: Convert Azure ARM templates and Bicep to Pulumi
- **pulumi-arm-import**: Import existing Azure resources into Pulumi
- **pulumi-cloudformation-id-lookup**: Look up CloudFormation import identifiers

### Authoring Plugin (`authoring/`)

Skills for writing quality Pulumi programs, components, automation, and secrets management:

- **pulumi-best-practices**: Best practices for writing reliable Pulumi programs
- **pulumi-component**: Guide for authoring ComponentResource classes
- **pulumi-automation-api**: Best practices for using Pulumi Automation API
- **esc**: Guidance for working with Pulumi ESC (Environments, Secrets, and Configuration)

## Installation

### Claude Code Plugin System

```bash
/plugin marketplace add pulumi/agent-skills
/plugin install pulumi-migration
/plugin install pulumi-authoring
```

### Universal (all agents)

Install all skills:

```bash
npx skills add pulumi/agent-skills
```

Or install individual plugin groups:

```bash
npx skills add pulumi/agent-skills/migration      # 7 migration skills
npx skills add pulumi/agent-skills/authoring      # 4 authoring skills
```

This works with Claude Code, Cursor, Copilot, Codex, and other agent tools.

## Skill Writing Conventions

### Cross-Skill References

When one skill references another, use the pattern: `Use skill <skill-name>`.

Examples:
- "Use skill `pulumi-arm-import` for zero-diff import validation"
- "Use skill `pulumi-component` for in-depth component authoring guidance"

### Skill Name Format

Skill names follow the pattern: `pulumi-<feature>` or just `<feature>` for short names like `esc`.

### Trigger Phrases

Skill descriptions should include keywords and phrases that trigger automatic activation. Use "MUST be loaded when" or "Use when" to clarify activation criteria.

Example:
```yaml
description: Convert an AWS CDK application to Pulumi. This skill MUST be loaded whenever a user requests migration or conversion of a CDK application to Pulumi.
```

### Progressive Disclosure

Keep the main SKILL.md file focused and concise (under 500 lines recommended). For detailed reference material, use a `references/` subdirectory with specific topic files.

## Adding a New Skill

1. Determine which plugin group the skill belongs to (migration or authoring)
2. Create the skill directory: `<plugin>/skills/<skill-name>/`
3. Write the `SKILL.md` file with proper frontmatter:
   ```yaml
   ---
   name: skill-name
   description: Clear description with activation triggers
   ---
   ```
4. Update this AGENTS.md file to list the new skill in the appropriate plugin section
5. Update [README.md](README.md) to add the skill to the skills table
6. Submit a pull request

The skill will automatically be included in its plugin group. No manifest updates are needed.

## Creating a New Plugin Group

1. Create the plugin directory structure:
   ```
   <plugin-name>/
   ├── .claude-plugin/
   │   └── plugin.json
   └── skills/
   ```

2. Create `plugin.json`:
   ```json
   {
     "name": "pulumi-<plugin-name>",
     "version": "1.0.0",
     "description": "Plugin description",
     "author": { "name": "Pulumi", "url": "https://github.com/pulumi" },
     "homepage": "https://www.pulumi.com/docs/",
     "repository": "https://github.com/pulumi/agent-skills",
     "license": "Apache-2.0",
     "keywords": ["pulumi", "..."],
     "mcpServers": {
       "pulumi": {
         "command": "npx",
         "args": ["-y", "@pulumi/mcp-server@latest"],
         "env": { "PULUMI_ACCESS_TOKEN": "${PULUMI_ACCESS_TOKEN}" }
       }
     }
   }
   ```

3. Add an entry to [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json):
   ```json
   {
     "name": "pulumi-<plugin-name>",
     "source": "./<plugin-name>",
     "description": "Plugin description",
     "category": "appropriate-category"
   }
   ```

4. Add skills to `<plugin-name>/skills/`
5. Update this AGENTS.md and README.md
6. Submit a pull request

## MCP Server Integration

All plugins bundle the Pulumi MCP server configuration. When you install a plugin through Claude Code, the MCP server is automatically configured, providing:

- Access to your Pulumi Cloud organization data
- Cross-stack visibility through Pulumi Insights
- Live infrastructure context

The MCP server requires the `PULUMI_ACCESS_TOKEN` environment variable. Set this in your shell profile or Claude Code settings.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed contribution guidelines, including skill quality standards, formatting requirements, and the review process.
