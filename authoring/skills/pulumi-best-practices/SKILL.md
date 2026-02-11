---
name: pulumi-best-practices
version: 1.0.0
description: Best practices for writing reliable Pulumi programs. Covers Output handling, resource dependencies, component structure, secrets management, safe refactoring with aliases, and deployment workflows.
---

# Pulumi Best Practices

## When to Use This Skill

Invoke this skill when:

- Writing new Pulumi programs or components
- Reviewing Pulumi code for correctness
- Refactoring existing Pulumi infrastructure
- Debugging resource dependency issues
- Setting up configuration and secrets

## Practices

### 1. Never Create Resources Inside Apply/Transform Callbacks

**Why**: Resources created inside apply/transform callbacks don't appear in `pulumi preview`, making changes unpredictable. Pulumi cannot properly track dependencies, leading to race conditions and deployment failures.

**Detection signals**:

- Resource constructors inside apply/transform callbacks
- Resource creation inside combined-output apply calls
- Dynamic resource counts determined at runtime inside apply

**When apply is appropriate**:

- Transforming output values for use in tags, names, or computed strings
- Logging or debugging (not resource creation)
- Conditional logic that affects resource properties, not resource existence

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

**Reference**: https://www.pulumi.com/docs/concepts/inputs-outputs/

---

### 2. Pass Outputs Directly as Inputs

**Why**: Pulumi builds a directed acyclic graph (DAG) based on input/output relationships. Passing outputs directly ensures correct creation order. Unwrapping values manually breaks the dependency chain, causing resources to deploy in wrong order or reference values that don't exist yet.

**Detection signals**:

- Variables extracted from apply/transform callbacks used later as resource inputs
- String concatenation with outputs instead of using Pulumi's interpolation helpers
- Manually unwrapping output values and passing raw strings/values to resources

**For string interpolation**, use Pulumi's built-in interpolation helpers rather than manually applying transformations. Each language SDK provides concatenation and formatting utilities that preserve the dependency graph.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

**Reference**: https://www.pulumi.com/docs/concepts/inputs-outputs/

---

### 3. Use Components for Related Resources

**Why**: Component resources group related resources into reusable, logical units. Without components, your resource graph is flat, making it hard to understand which resources belong together, reuse patterns across stacks, or reason about your infrastructure at a higher level.

**Detection signals**:

- Multiple related resources created at top level without grouping
- Repeated resource patterns across stacks that should be abstracted
- Hard to understand resource relationships from the Pulumi console

**Component best practices**:

- Use a consistent type URN pattern: `organization:module:ComponentName`
- Call the register-outputs method at the end of the constructor
- Expose outputs as properties/fields for consumers
- Accept resource options to allow callers to set providers, aliases, etc.

For in-depth component authoring guidance (args design, multi-language support, testing, distribution), use skill `pulumi-component`.

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

**Reference**: https://www.pulumi.com/docs/concepts/resources/components/

---

### 4. Always Set the Parent in Components

**Why**: When you create resources inside a component without setting the parent, those resources appear at the root level of your stack's state. This breaks the logical hierarchy, makes the Pulumi console hard to navigate, and can cause issues with aliases and refactoring. The parent relationship is what makes the component actually group its children.

**Detection signals**:

- Component constructors that don't set the parent on child resources
- Resources inside a component appearing at root level in the console
- Unexpected behavior when adding aliases to components

**What setting the parent provides**:

- Resources appear nested under the component in Pulumi console
- Deleting the component deletes all children
- Aliases on the component automatically apply to children
- Clear ownership in state files

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

**Reference**: https://www.pulumi.com/docs/concepts/resources/components/

---

### 5. Encrypt Secrets from Day One

**Why**: Secrets marked with `--secret` are encrypted in state files, masked in CLI output, and tracked through transformations. Starting with plaintext config and converting later requires credential rotation, reference updates, and audit of leaked values in logs and state history.

**Detection signals**:

- Passwords, API keys, tokens stored as plain config
- Connection strings with embedded credentials
- Private keys or certificates in plaintext

**Wrong**:

```bash
# Plaintext - will be visible in state and logs
pulumi config set databasePassword hunter2
pulumi config set apiKey sk-1234567890
```

**Right**:

```bash
# Encrypted from the start
pulumi config set --secret databasePassword hunter2
pulumi config set --secret apiKey sk-1234567890
```

**Use Pulumi ESC for centralized secrets**:

```yaml
# Pulumi.yaml
environment:
  - production-secrets  # Pull from ESC environment
```

```bash
# ESC manages secrets centrally across stacks
esc env set production-secrets db.password --secret "hunter2"
```

**What qualifies as a secret**:

- Passwords and passphrases
- API keys and tokens
- Private keys and certificates
- Connection strings with credentials
- OAuth client secrets
- Encryption keys

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

**References**:

- https://www.pulumi.com/docs/concepts/secrets/
- https://www.pulumi.com/docs/esc/

---

### 6. Use Aliases When Refactoring

**Why**: Renaming resources, moving them into components, or changing parents causes Pulumi to see them as new resources. Without aliases, refactoring destroys and recreates resources, potentially causing downtime or data loss. Aliases preserve resource identity through refactors.

**Detection signals**:

- Resource rename without alias
- Moving resource into or out of a component
- Changing the parent of a resource
- Preview shows delete+create when update was intended

**Alias types**:

- Simple name change — alias with the old name
- Parent change — alias with old name and old parent
- Full URN — when you know the exact previous URN

**Lifecycle**:

1. Add alias during refactor
2. Run `pulumi up` on all stacks
3. Remove alias after all stacks updated (optional, but keeps code clean)

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

**Reference**: https://www.pulumi.com/docs/iac/concepts/resources/options/aliases/

---

### 7. Preview Before Every Deployment

**Why**: `pulumi preview` shows exactly what will be created, updated, or destroyed. Surprises in production come from skipping preview. A resource showing "replace" when you expected "update" means imminent destruction and recreation.

**Detection signals**:

- Running `pulumi up --yes` interactively without reviewing changes
- No preview step anywhere in the CI/CD workflow for a given change
- Preview output not reviewed before merge or deployment approval

**Wrong**:

```bash
# Deploying blind
pulumi up --yes
```

**Right**:

```bash
# Always preview first
pulumi preview

# Review the output, then deploy
pulumi up
```

**What to look for in preview**:

- `+ create` - New resource will be created
- `~ update` - Existing resource will be modified in place
- `- delete` - Resource will be destroyed
- `+-replace` - Resource will be destroyed and recreated (potential downtime)
- `~+-replace` - Resource will be updated, then replaced

**Warning signs**:

- Unexpected `replace` operations (check for immutable property changes)
- Resources being deleted that shouldn't be
- More changes than expected from your code diff

**CI/CD integration**:

```yaml
# GitHub Actions example
jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Pulumi Preview
        uses: pulumi/actions@v5
        with:
          command: preview
          stack-name: production
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}

  deploy:
    needs: preview
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Pulumi Up
        uses: pulumi/actions@v5
        with:
          command: up
          stack-name: production
```

**PR workflow**:

- Run preview on every PR
- Post preview output as PR comment
- Require preview review before merge
- Deploy only on merge to main

**References**:

- https://www.pulumi.com/docs/cli/commands/pulumi_preview/
- https://www.pulumi.com/docs/iac/packages-and-automation/continuous-delivery/github-actions/

---

## Quick Reference

| Practice | Key Signal | Fix |
|----------|-----------|-----|
| No resources in apply | Resource constructors inside apply callbacks | Move resource outside, pass Output directly |
| Pass outputs directly | Extracted values used as inputs | Use Output objects, Pulumi's interpolation helpers |
| Use components | Flat structure, repeated patterns | Create component resource classes/structs |
| Set parent | Component children at root level | Set the parent on all child resources |
| Secrets from day one | Plaintext passwords/keys in config | Use `--secret` flag, ESC |
| Aliases when refactoring | Delete+create in preview | Add alias with old name/parent |
| Preview before deploy | `pulumi up --yes` | Always run `pulumi preview` first |

## Validation Checklist

When reviewing Pulumi code, verify:

- [ ] No resource constructors inside apply/transform callbacks
- [ ] Outputs passed directly to dependent resources
- [ ] Related resources grouped in component resource classes
- [ ] Child resources have the parent set to the component
- [ ] Sensitive values use secret config retrieval or `--secret`
- [ ] Refactored resources have aliases preserving identity
- [ ] Deployment process includes preview step

## Related Skills

- **pulumi-component**: Deep guide to authoring component resources, designing args, multi-language support, testing, and distribution. Use skill `pulumi-component`.
- **pulumi-automation-api**: Programmatic orchestration of multiple stacks. Use skill `pulumi-automation-api`.
- **pulumi-esc**: Centralized secrets and configuration management. Use skill `pulumi-esc`.
