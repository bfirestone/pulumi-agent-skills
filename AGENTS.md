# Pulumi Agent Skills

This repository contains official Pulumi agent skills for infrastructure as code workflows. Skills support **TypeScript**, **Go**, and **Python**.

## Multi-Language Support

Each skill uses a progressive disclosure pattern:

- **`SKILL.md`** contains language-neutral concepts, workflows, and decision guidance
- **`examples-ts.md`** contains TypeScript code examples and patterns
- **`examples-go.md`** contains Go code examples and patterns
- **`examples-python.md`** contains Python code examples and patterns

**When the user specifies a language** (TypeScript, Go, or Python), load the corresponding `examples-{lang}.md` file for that skill. **If no language is specified**, ask the user which language they are using before loading examples.

## Repository Structure

Skills are organized into two plugin groups:

### Migration Plugin (`migration/`)

Skills for converting and importing infrastructure from other tools to Pulumi:

- **pulumi-terraform-to-pulumi**: Migrate Terraform projects to Pulumi
- **pulumi-cdk-to-pulumi**: Migrate AWS CDK applications to Pulumi
- **cloudformation-to-pulumi**: Migrate AWS CloudFormation stacks/templates to Pulumi
- **pulumi-arm-to-pulumi**: Migrate Azure ARM templates and Bicep to Pulumi

### Authoring Plugin (`authoring/`)

Skills for writing quality Pulumi programs, components, automation, and secrets management:

- **pulumi-best-practices**: Best practices for writing reliable Pulumi programs
- **pulumi-component**: Guide for authoring ComponentResource classes
- **pulumi-automation-api**: Best practices for using Pulumi Automation API
- **pulumi-esc**: Guidance for working with Pulumi ESC (Environments, Secrets, and Configuration)

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
npx skills add pulumi/agent-skills --skill '*'
```

Or install individual plugin groups:

```bash
npx skills add pulumi/agent-skills/migration --skill '*'     # 4 migration skills
npx skills add pulumi/agent-skills/authoring --skill '*'     # 4 authoring skills
```

This works with Claude Code, Cursor, Copilot, Codex, and other agent tools.

## Skill Writing Conventions

### Skill Name Format

Skill names follow the pattern: `pulumi-<feature>`.

### Trigger Phrases

Skill descriptions should include keywords and phrases that trigger automatic activation. Use "MUST be loaded when" or "Use when" to clarify activation criteria.

Example:
```yaml
description: Convert an AWS CDK application to Pulumi. This skill MUST be loaded whenever a user requests migration or conversion of a CDK application to Pulumi.
```

### Progressive Disclosure

Each skill directory uses this structure:

```
skill-name/
├── SKILL.md              # Language-neutral concepts, workflows, decisions
├── examples-ts.md        # TypeScript code examples
├── examples-go.md        # Go code examples
├── examples-python.md    # Python code examples
└── *.md                  # Additional reference files (tool usage, lookups, etc.)
```

The main `SKILL.md` file should be focused and concise (under 500 lines recommended). It contains:
- Workflow steps and decision points
- Tool commands and CLI usage
- Conceptual guidance that applies regardless of language

Language-specific examples files (`examples-{lang}.md`) contain:
- Code snippets showing idiomatic patterns for that language
- Before/after transformation examples
- Language-specific imports, types, and error handling

Reference additional files from `SKILL.md` using relative links:
```markdown
For detailed tool usage, see [cdk-convert.md](cdk-convert.md).
```

### Cross-Skill References

When one skill references another, use the pattern: `Use skill <skill-name>`.

Example:
- "Use skill `pulumi-component` for in-depth component authoring guidance"

## Language-Specific Notes

### TypeScript

- Use `async/await` for all asynchronous operations
- Import from `@pulumi/pulumi`, `@pulumi/aws`, etc.
- Use `pulumi.Output` for output values
- Project setup: `pulumi new typescript`

### Go

- Always handle errors with `if err != nil { return err }`
- Import from `github.com/pulumi/pulumi/sdk/v3/go/pulumi`, etc.
- Use `pulumi.String()`, `pulumi.Int()` for input wrappers
- Project setup: `pulumi new go`

### Python

- Use `snake_case` for all resource arguments and property names
- Import from `pulumi`, `pulumi_aws`, etc.
- Use `pulumi.Output` for output values
- Project setup: `pulumi new python`

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
4. Write `examples-ts.md`, `examples-go.md`, and `examples-python.md` with language-specific code
5. Update this AGENTS.md file to list the new skill in the appropriate plugin section
6. Update [README.md](README.md) to add the skill to the skills table
7. Submit a pull request

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
     "keywords": ["pulumi", "..."]
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

4. Add skills to `<plugin-name>/skills/` with `SKILL.md` and all three `examples-{lang}.md` files
5. Update this AGENTS.md and README.md
6. Submit a pull request

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed contribution guidelines, including skill quality standards, formatting requirements, and the review process.
