---
name: pulumi-automation-api
version: 1.0.0
description: Best practices for using Pulumi Automation API to programmatically orchestrate infrastructure operations. Covers multi-stack orchestration, embedding Pulumi in applications, architecture choices, and common patterns.
---

# Pulumi Automation API

## When to Use This Skill

Invoke this skill when:

- Orchestrating deployments across multiple Pulumi stacks
- Embedding Pulumi operations in custom applications
- Building self-service infrastructure platforms
- Replacing fragile Bash/Makefile orchestration scripts
- Creating custom CLIs for infrastructure management
- Building web applications that provision infrastructure

## What is Automation API

Automation API provides programmatic access to Pulumi operations. Instead of running `pulumi up` from the CLI, you call functions in your code that perform the same operations. You create or select a stack, then run operations like `up`, `preview`, or `destroy` programmatically.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

## When to Use Automation API

### Good Use Cases

**Multi-stack orchestration:**

When you split infrastructure into multiple focused projects, Automation API helps offset the added complexity by orchestrating operations across stacks:

```text
infrastructure → platform → application
     ↓              ↓            ↓
   (VPC)      (Kubernetes)   (Services)
```

Automation API ensures correct sequencing without manual intervention.

**Self-service platforms:**

Build internal tools where developers request infrastructure without learning Pulumi:

- Web portals for environment provisioning
- Slack bots that create/destroy resources
- Custom CLIs tailored to your organization

**Embedded infrastructure:**

Applications that provision their own infrastructure:

- SaaS platforms creating per-tenant resources
- Testing frameworks spinning up test environments
- CI/CD systems with dynamic infrastructure needs

**Replacing fragile scripts:**

If you have Bash scripts or Makefiles stitching together multiple `pulumi` commands, Automation API provides:

- Proper error handling
- Type safety
- Programmatic access to outputs

### When NOT to Use

- Single project with standard deployment needs
- When you don't need programmatic control over operations

## Architecture Choices

### Local Source vs Inline Source

**Local Source** - Pulumi program in separate files. You point the workspace at an existing Pulumi project directory.

**When to use:**

- Different teams maintain orchestrator vs Pulumi programs
- Pulumi programs already exist
- Want independent version control and release cycles
- Platform team orchestrating application team's infrastructure

**Inline Source** - Pulumi program embedded in the orchestrator code. You define the infrastructure program as a function passed directly to the stack constructor.

**When to use:**

- Single team owns everything
- Tight coupling between orchestration and infrastructure is desired
- Distributing as a compiled binary or packaged application (no source files needed)
- Simpler deployment artifact

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Language Independence

The Automation API program can use a different language than the Pulumi programs it orchestrates:

```text
Orchestrator (any language) → manages → Pulumi Program (any language)
```

This enables platform teams to use their preferred language while application teams use theirs.

## Common Patterns

### Multi-Stack Orchestration

Deploy multiple stacks in dependency order. Define the list of stacks with their directories, then iterate through them sequentially, creating or selecting each stack and running `up`. For teardown, destroy in reverse order.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Passing Configuration

Set stack configuration programmatically before deploying. You can set both plain values and secrets on a stack using the configuration API.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Reading Outputs

Access stack outputs after deployment. Outputs are available both from the operation result and by querying the stack directly.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Error Handling

Handle deployment failures gracefully. Check both for exceptions/errors from the operation and inspect the result summary for failure status.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Parallel Stack Operations

When stacks are independent, deploy them in parallel using language-appropriate concurrency primitives.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

## Best Practices

### Separate Configuration from Code

Externalize configuration into files or environment variables. Load deployment configuration (stack names, directories, environment) from a JSON file or similar, then iterate over the entries. This enables distributing compiled binaries or packaged applications without exposing source code.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Stream Output for Long Operations

Use the output streaming callback/writer for real-time feedback during long-running operations. This output can be directed to stdout, a logging system, a websocket, or any other destination.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

## Quick Reference

| Scenario | Approach |
| --- | --- |
| Existing Pulumi projects | Local source with work directory |
| New embedded infrastructure | Inline source with program function |
| Different teams | Local source for independence |
| Packaged/compiled distribution | Inline source or bundled local |
| Multi-stack dependencies | Sequential deployment in order |
| Independent stacks | Parallel deployment with concurrency primitives |

> See language-specific examples for API names: [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

## Related Skills

- **pulumi-best-practices**: Code-level patterns for Pulumi programs

## References

- https://www.pulumi.com/docs/using-pulumi/automation-api/
- https://www.pulumi.com/docs/using-pulumi/automation-api/concepts-terminology/
- https://www.pulumi.com/blog/iac-recommended-practices-using-automation-api/
