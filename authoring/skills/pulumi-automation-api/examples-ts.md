# TypeScript Examples: Pulumi Automation API

## What is Automation API

Basic usage — create or select a stack and run `up`:

```typescript
import * as automation from "@pulumi/pulumi/automation";

// Create or select a stack
const stack = await automation.LocalWorkspace.createOrSelectStack({
    stackName: "dev",
    projectName: "my-project",
    program: async () => {
        // Your Pulumi program here
    },
});

// Run pulumi up programmatically
const upResult = await stack.up({ onOutput: console.log });
console.log(`Update summary: ${JSON.stringify(upResult.summary)}`);
```

## Architecture Choices

### Local Source

```typescript
const stack = await automation.LocalWorkspace.createOrSelectStack({
    stackName: "dev",
    workDir: "./infrastructure",  // Points to existing Pulumi project
});
```

### Inline Source

```typescript
import * as aws from "@pulumi/aws";

const stack = await automation.LocalWorkspace.createOrSelectStack({
    stackName: "dev",
    projectName: "my-project",
    program: async () => {
        const bucket = new aws.s3.Bucket("my-bucket");
        return { bucketName: bucket.id };
    },
});
```

## Common Patterns

### Multi-Stack Orchestration

Deploy multiple stacks in dependency order:

```typescript
import * as automation from "@pulumi/pulumi/automation";

async function deploy() {
    const stacks = [
        { name: "infrastructure", dir: "./infra" },
        { name: "platform", dir: "./platform" },
        { name: "application", dir: "./app" },
    ];

    for (const stackInfo of stacks) {
        console.log(`Deploying ${stackInfo.name}...`);

        const stack = await automation.LocalWorkspace.createOrSelectStack({
            stackName: "prod",
            workDir: stackInfo.dir,
        });

        await stack.up({ onOutput: console.log });
        console.log(`${stackInfo.name} deployed successfully`);
    }
}

async function destroy() {
    // Destroy in reverse order
    const stacks = [
        { name: "application", dir: "./app" },
        { name: "platform", dir: "./platform" },
        { name: "infrastructure", dir: "./infra" },
    ];

    for (const stackInfo of stacks) {
        console.log(`Destroying ${stackInfo.name}...`);

        const stack = await automation.LocalWorkspace.selectStack({
            stackName: "prod",
            workDir: stackInfo.dir,
        });

        await stack.destroy({ onOutput: console.log });
    }
}
```

### Passing Configuration

```typescript
const stack = await automation.LocalWorkspace.createOrSelectStack({
    stackName: "dev",
    workDir: "./infrastructure",
});

// Set configuration values
await stack.setConfig("aws:region", { value: "us-west-2" });
await stack.setConfig("dbPassword", { value: "secret", secret: true });

// Then deploy
await stack.up();
```

### Reading Outputs

```typescript
const upResult = await stack.up();

// Get all outputs
const outputs = await stack.outputs();
console.log(`VPC ID: ${outputs["vpcId"].value}`);

// Or from the up result
console.log(`Outputs: ${JSON.stringify(upResult.outputs)}`);
```

### Error Handling

```typescript
try {
    const result = await stack.up({ onOutput: console.log });

    if (result.summary.result === "failed") {
        console.error("Deployment failed");
        process.exit(1);
    }
} catch (error) {
    console.error(`Deployment error: ${error}`);
    throw error;
}
```

### Parallel Stack Operations

```typescript
const independentStacks = [
    { name: "service-a", dir: "./service-a" },
    { name: "service-b", dir: "./service-b" },
    { name: "service-c", dir: "./service-c" },
];

await Promise.all(independentStacks.map(async (stackInfo) => {
    const stack = await automation.LocalWorkspace.createOrSelectStack({
        stackName: "prod",
        workDir: stackInfo.dir,
    });
    return stack.up({ onOutput: (msg) => console.log(`[${stackInfo.name}] ${msg}`) });
}));
```

## Best Practices

### Separate Configuration from Code

```typescript
import * as fs from "fs";

interface DeployConfig {
    stacks: Array<{ name: string; dir: string; }>;
    environment: string;
}

const config: DeployConfig = JSON.parse(
    fs.readFileSync("./deploy-config.json", "utf-8")
);

for (const stackInfo of config.stacks) {
    const stack = await automation.LocalWorkspace.createOrSelectStack({
        stackName: config.environment,
        workDir: stackInfo.dir,
    });
    await stack.up();
}
```

### Stream Output for Long Operations

```typescript
await stack.up({
    onOutput: (message) => {
        process.stdout.write(message);
        // Or send to logging system, websocket, etc.
    },
});
```

## Quick Reference

| Scenario | Approach |
| --- | --- |
| Existing Pulumi projects | Local source with `workDir` |
| New embedded infrastructure | Inline source with `program` function |
| Different teams | Local source for independence |
| Compiled binary distribution | Inline source or bundled local |
| Multi-stack dependencies | Sequential deployment in order |
| Independent stacks | Parallel deployment with `Promise.all` |
| Create/select stack | `automation.LocalWorkspace.createOrSelectStack()` |
| Select existing stack | `automation.LocalWorkspace.selectStack()` |
| Set config | `stack.setConfig(key, { value, secret? })` |
| Get outputs | `stack.outputs()` |
| Stream output | `stack.up({ onOutput: callback })` |
