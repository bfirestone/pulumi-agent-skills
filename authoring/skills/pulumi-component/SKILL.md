---
name: pulumi-component
version: 1.0.0
description: Guide for authoring Pulumi ComponentResource classes. Use when creating reusable infrastructure components, designing component interfaces, setting up multi-language support, or distributing component packages.
---

# Authoring Pulumi Components

A ComponentResource groups related infrastructure resources into a reusable, logical unit. Components make infrastructure easier to understand, reuse, and maintain. Components appear as a single node with children nested underneath in `pulumi preview`/`pulumi up` output and in the Pulumi Cloud console.

This skill covers the full component authoring lifecycle. For general Pulumi coding patterns (Output handling, secrets, aliases, preview workflows), use the `pulumi-best-practices` skill instead.

## When to Use This Skill

Invoke this skill when:

- Creating a new component resource
- Designing the args interface for a component
- Making a component consumable from multiple Pulumi languages
- Publishing or distributing a component package
- Refactoring inline resources into a reusable component
- Debugging component behavior (missing outputs, stuck creating, children at wrong level)

## Component Anatomy

Every component has four required elements:

1. **Extend the ComponentResource base** and register with a type URN
2. **Accept standard parameters**: name, args, and component resource options
3. **Set the parent** on all child resources
4. **Register outputs** at the end of the constructor

> **Code examples:** [TypeScript](examples-ts.md) | [Go](examples-go.md) | [Python](examples-python.md)

### Type URN Format

The type URN passed when registering the component follows the format: `<package>:<module>:<type>`.

| Segment | Convention | Example |
|---------|-----------|---------|
| package | Organization or package name | `myorg`, `acme`, `pkg` |
| module  | Usually `index` | `index` |
| type    | PascalCase type name | `StaticSite`, `VpcNetwork` |

Full examples: `myorg:index:StaticSite`, `acme:index:KubernetesCluster`

### Registering Outputs Is Required

**Why**: Without registering outputs, the component appears stuck in a "creating" state in the Pulumi console and outputs are not persisted to state.

> **Code examples:** [TypeScript](examples-ts.md#registering-outputs-is-required) | [Go](examples-go.md#registering-outputs-is-required) | [Python](examples-python.md#registering-outputs-is-required)

### Derive Child Names from the Component Name

**Why**: Hardcoded child names cause collisions when the component is instantiated multiple times.

> **Code examples:** [TypeScript](examples-ts.md#derive-child-names-from-the-component-name) | [Go](examples-go.md#derive-child-names-from-the-component-name) | [Python](examples-python.md#derive-child-names-from-the-component-name)

---

## Designing the Args Interface

The args interface is the most impactful design decision. It defines what consumers can configure and how composable the component is.

### Wrap Properties in Input Types

**Why**: Input types accept both plain values and outputs from other resources. Without them, consumers must unwrap outputs manually with apply.

> **Code examples:** [TypeScript](examples-ts.md#wrap-properties-in-input-types) | [Go](examples-go.md#wrap-properties-in-input-types) | [Python](examples-python.md#wrap-properties-in-input-types)

### Keep Structures Flat

Avoid deeply nested arg objects. Flat structures are easier to use and evolve.

> **Code examples:** [TypeScript](examples-ts.md#keep-structures-flat) | [Go](examples-go.md#keep-structures-flat) | [Python](examples-python.md#keep-structures-flat)

### No Union Types

Union types break multi-language SDK generation. Not all languages can represent union types.

If you need to accept multiple forms, use separate optional properties.

> **Code examples:** [TypeScript](examples-ts.md#no-union-types) | [Go](examples-go.md#no-union-types) | [Python](examples-python.md#no-union-types)

### No Functions or Callbacks

Functions cannot be serialized across language boundaries.

> **Code examples:** [TypeScript](examples-ts.md#no-functions-or-callbacks) | [Go](examples-go.md#no-functions-or-callbacks) | [Python](examples-python.md#no-functions-or-callbacks)

### Use Defaults for Optional Properties

Set sensible defaults inside the constructor so consumers only configure what they need.

> **Code examples:** [TypeScript](examples-ts.md#use-defaults-for-optional-properties) | [Go](examples-go.md#use-defaults-for-optional-properties) | [Python](examples-python.md#use-defaults-for-optional-properties)

---

## Exposing Outputs

### Expose Only What Consumers Need

Components often create many internal resources. Expose only the values consumers need, not every internal resource.

> **Code examples:** [TypeScript](examples-ts.md#expose-only-what-consumers-need) | [Go](examples-go.md#expose-only-what-consumers-need) | [Python](examples-python.md#expose-only-what-consumers-need)

### Derive Composite Outputs

Use output interpolation or composition to build derived values like connection strings.

> **Code examples:** [TypeScript](examples-ts.md#derive-composite-outputs) | [Go](examples-go.md#derive-composite-outputs) | [Python](examples-python.md#derive-composite-outputs)

---

## Component Design Patterns

### Sensible Defaults with Override

Encode best practices as defaults. Allow consumers to override when they have specific requirements.

> **Code examples:** [TypeScript](examples-ts.md#sensible-defaults-with-override) | [Go](examples-go.md#sensible-defaults-with-override) | [Python](examples-python.md#sensible-defaults-with-override)

### Conditional Resource Creation

Use optional args to gate creation of sub-resources.

> **Code examples:** [TypeScript](examples-ts.md#conditional-resource-creation) | [Go](examples-go.md#conditional-resource-creation) | [Python](examples-python.md#conditional-resource-creation)

### Composition

Build higher-level components from lower-level ones. Each level manages a single concern.

> **Code examples:** [TypeScript](examples-ts.md#composition) | [Go](examples-go.md#composition) | [Python](examples-python.md#composition)

### Provider Passthrough

Accept explicit providers for multi-region or multi-account deployments. Component resource options carry provider configuration to children automatically.

Children with the parent set to the component automatically inherit the provider. No extra code is needed inside the component.

> **Code examples:** [TypeScript](examples-ts.md#provider-passthrough) | [Go](examples-go.md#provider-passthrough) | [Python](examples-python.md#provider-passthrough)

---

## Multi-Language Components

If your component will be consumed from multiple Pulumi languages (TypeScript, Python, Go, C#, Java, YAML), package it as a multi-language component.

### Do You Need Multi-Language?

Ask: "Will anyone consume this component from a different language than it was authored in?"

**Single-language component** (no packaging needed):

- Your team uses one language and the component stays within that codebase
- The component is internal to a single project or monorepo
- No `PulumiPlugin.yaml` needed -- just import directly

**Multi-language component** (packaging required):

- Other teams consume your component in different languages
- Platform teams building abstractions for developers who choose their own language
- YAML consumers need access -- YAML programs require multi-language packaging to use your component
- Building a shared component library for your organization
- Publishing to the Pulumi private registry or public registry is a common reason, but not required for multi-language support

**Common mistake**: A platform team builds components only their own language users can consume. If application developers use other languages or YAML, those components are invisible to them without multi-language packaging.

### Setup

Create a `PulumiPlugin.yaml` in the component directory to declare the runtime:

```yaml
runtime: nodejs  # or python, go, dotnet
```

### Serialization Constraints

For multi-language compatibility, args must be serializable. These constraints apply regardless of the authoring language:

| Allowed | Not Allowed |
|---------|-------------|
| Primitive types (string, number/int, boolean) | Union types |
| Input type wrappers | Functions and callbacks |
| Arrays and maps of primitives | Complex nested generics |
| Enums | Platform-specific types |

### Consuming Multi-Language Components

Consumers install the component with `pulumi package add`, which automatically downloads the provider plugin, generates a local SDK in the consumer's language, and updates `Pulumi.yaml`:

```bash
# From a Git repository
pulumi package add <git-repo-url>

# From a specific version tag
pulumi package add <git-repo-url>@v1.0.0
```

For fresh checkouts or CI environments, run `pulumi install` to ensure all package dependencies are available. The consumer does not need to manually generate SDKs.

Authors who publish SDKs to package managers (npm, PyPI, etc.) can optionally use `pulumi package gen-sdk` to generate language-specific SDKs for publishing. Most component authors do not need this -- `pulumi package add` handles SDK generation on the consumer side.

### Entry Points

Published multi-language components require an entry point that hosts the component provider process. The entry point pattern differs by authoring language.

> **Code examples:** [TypeScript](examples-ts.md#entry-points) | [Go](examples-go.md#entry-points) | [Python](examples-python.md#entry-points)

For a complete working example across all languages, see https://github.com/mikhailshilkov/comp-as-comp.

**Reference**: https://www.pulumi.com/docs/iac/using-pulumi/pulumi-packages/

---

## Distribution

Choose a distribution method based on your audience:

| Audience | Method | How |
|----------|--------|-----|
| Same project | Direct import | Standard language import |
| Same organization | Private registry | `pulumi package publish` to Pulumi Cloud |
| Same organization | Git repository | `pulumi package add <repo>` with version tags |
| Language ecosystem | Package manager | Publish to npm, PyPI, NuGet, or Maven |
| Public community | Pulumi Registry | Submit via pulumi/registry GitHub repo |

### Pulumi Private Registry

The private registry is the centralized catalog for your organization's components. It provides automatic API documentation, version management, and discoverability for all teams.

Publish a component to the private registry:

```bash
pulumi package publish https://github.com/myorg/my-component --publisher myorg
```

Version components using git tags with a `v` prefix:

```bash
git tag v1.0.0
git push origin v1.0.0
```

A README file is required when publishing. Pulumi uses it as the component's documentation page in the registry.

Automate publishing from GitHub Actions using OIDC authentication:

```yaml
name: Publish Component
on:
  push:
    tags:
      - "v*"

permissions:
  id-token: write
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    env:
      PULUMI_ORG: myorg
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: pulumi/auth-actions@v1
        with:
          organization: ${{ env.PULUMI_ORG }}
          requested-token-type: urn:pulumi:token-type:access_token:organization
      - run: pulumi package publish https://github.com/${{ github.repository }} --publisher ${{ env.PULUMI_ORG }}
```

**Prerequisites**: Configure GitHub OIDC integration with Pulumi Cloud before using this workflow.

The registry supports private GitHub and GitLab repositories. For non-OIDC setups, authenticate with `GITHUB_TOKEN` or `GITLAB_TOKEN` environment variables.

The private registry automatically generates SDK documentation for each published component. Enrich the generated docs by adding type annotations and documentation to your component's inputs and outputs.

**Reference**: https://www.pulumi.com/docs/idp/get-started/private-registry/

### Git Repository Distribution

Tag releases for consumers to pin versions:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Consumers install with:

```bash
pulumi package add https://github.com/myorg/my-component@v1.0.0
```

### Package Manager Distribution

Publish language-specific packages for native dependency management:

- **npm**: `npm publish` for TypeScript/JavaScript
- **PyPI**: `twine upload` for Python
- **NuGet**: `dotnet nuget push` for .NET
- **Maven Central**: Standard Maven publishing for Java

**Reference**: https://www.pulumi.com/docs/iac/using-pulumi/pulumi-packages/

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Resources inside apply | Not visible in `pulumi preview` | Move resource creation outside apply (see `pulumi-best-practices` practice 1) |
| Missing output registration | Component stuck "creating" | Always register outputs as last step of constructor |
| Missing parent option | Children appear at root level | Set parent to the component on all child resources |
| Union types in args | Breaks cross-language SDKs | Use single types; separate properties for variants |
| Functions in args | Cannot serialize across languages | Use configuration properties instead |
| Hardcoded child names | Collisions with multiple instances | Derive names from the component name with a suffix |
| Over-exposed outputs | Leaks implementation details | Export only what consumers need |
| Single-use component | Unnecessary abstraction overhead | Use inline resources until a pattern repeats |
| Deeply nested args | Hard to use and evolve | Keep structures flat with optional properties |

---

## Quick Reference

| Topic | Key Point |
|-------|-----------|
| Type URN | `<package>:<module>:<type>`, module usually `index` |
| Constructor | Register component with type URN, then create children, then register outputs |
| Child resources | Always set parent to component, derive name from component name + suffix |
| Args interface | Wrap in input types, no unions, no functions, flat structure |
| Outputs | Typed output properties, expose only essentials |
| Defaults | Apply sensible defaults in constructor for optional properties |
| Composition | Lower-level components composed into higher-level ones |
| Multi-language | `PulumiPlugin.yaml` + entry point; consumers use `pulumi package add` |
| Distribution | Private registry, git tags, package managers, or public Pulumi Registry |

## Related Skills

- **pulumi-best-practices**: General Pulumi patterns including Output handling, secrets, and aliases
- **pulumi-automation-api**: Programmatic orchestration for integration testing and multi-stack workflows
- **pulumi-esc**: Centralized secrets and configuration for component deployments

## References

- https://www.pulumi.com/docs/iac/concepts/resources/components/
- https://www.pulumi.com/docs/iac/using-pulumi/pulumi-packages/
- https://www.pulumi.com/docs/idp/get-started/private-registry/
- https://www.pulumi.com/docs/iac/concepts/inputs-outputs/
- https://www.pulumi.com/docs/iac/concepts/resources/options/aliases/
