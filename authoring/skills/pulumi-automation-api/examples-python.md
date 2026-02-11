# Python Examples: Pulumi Automation API

## What is Automation API

Basic usage — create or select a stack and run `up`:

```python
from pulumi import automation as auto

# Create or select a stack
def pulumi_program():
    # Your Pulumi program here
    pass

stack = auto.create_or_select_stack(
    stack_name="dev",
    project_name="my-project",
    program=pulumi_program,
)

# Run pulumi up programmatically
up_result = stack.up(on_output=print)
print(f"Update summary: {up_result.summary}")
```

## Architecture Choices

### Local Source

```python
from pulumi import automation as auto

stack = auto.create_or_select_stack(
    stack_name="dev",
    work_dir="./infrastructure",  # Points to existing Pulumi project
)
```

### Inline Source

```python
from pulumi import automation as auto
import pulumi_aws as aws

def pulumi_program():
    bucket = aws.s3.Bucket("my-bucket")
    return {"bucket_name": bucket.id}

stack = auto.create_or_select_stack(
    stack_name="dev",
    project_name="my-project",
    program=pulumi_program,
)
```

## Common Patterns

### Multi-Stack Orchestration

Deploy multiple stacks in dependency order:

```python
from pulumi import automation as auto


def deploy():
    stacks = [
        {"name": "infrastructure", "dir": "./infra"},
        {"name": "platform", "dir": "./platform"},
        {"name": "application", "dir": "./app"},
    ]

    for stack_info in stacks:
        print(f"Deploying {stack_info['name']}...")

        stack = auto.create_or_select_stack(
            stack_name="prod",
            work_dir=stack_info["dir"],
        )

        stack.up(on_output=print)
        print(f"{stack_info['name']} deployed successfully")


def destroy():
    # Destroy in reverse order
    stacks = [
        {"name": "application", "dir": "./app"},
        {"name": "platform", "dir": "./platform"},
        {"name": "infrastructure", "dir": "./infra"},
    ]

    for stack_info in stacks:
        print(f"Destroying {stack_info['name']}...")

        stack = auto.select_stack(
            stack_name="prod",
            work_dir=stack_info["dir"],
        )

        stack.destroy(on_output=print)
```

### Passing Configuration

```python
from pulumi import automation as auto

stack = auto.create_or_select_stack(
    stack_name="dev",
    work_dir="./infrastructure",
)

# Set configuration values
stack.set_config("aws:region", auto.ConfigValue(value="us-west-2"))
stack.set_config("dbPassword", auto.ConfigValue(value="secret", secret=True))

# Then deploy
stack.up()
```

### Reading Outputs

```python
import json

up_result = stack.up()

# Get all outputs
outputs = stack.outputs()
print(f"VPC ID: {outputs['vpcId'].value}")

# Or from the up result
print(f"Outputs: {json.dumps({k: v.value for k, v in up_result.outputs.items()})}")
```

### Error Handling

```python
import sys

try:
    result = stack.up(on_output=print)

    if result.summary.result == "failed":
        print("Deployment failed", file=sys.stderr)
        sys.exit(1)
except auto.errors.CommandError as e:
    print(f"Deployment error: {e}", file=sys.stderr)
    raise
```

### Parallel Stack Operations

```python
import concurrent.futures
from pulumi import automation as auto


def deploy_stack(stack_info: dict):
    stack = auto.create_or_select_stack(
        stack_name="prod",
        work_dir=stack_info["dir"],
    )
    return stack.up(on_output=lambda msg: print(f"[{stack_info['name']}] {msg}"))


independent_stacks = [
    {"name": "service-a", "dir": "./service-a"},
    {"name": "service-b", "dir": "./service-b"},
    {"name": "service-c", "dir": "./service-c"},
]

with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = [executor.submit(deploy_stack, s) for s in independent_stacks]
    results = [f.result() for f in concurrent.futures.as_completed(futures)]
```

## Best Practices

### Separate Configuration from Code

```python
import json
from pulumi import automation as auto

with open("./deploy-config.json") as f:
    config = json.load(f)

for stack_info in config["stacks"]:
    stack = auto.create_or_select_stack(
        stack_name=config["environment"],
        work_dir=stack_info["dir"],
    )
    stack.up()
```

### Stream Output for Long Operations

```python
stack.up(
    on_output=lambda message: print(message, end=""),
    # Or send to logging system, websocket, etc.
)
```

## Quick Reference

| Scenario | Approach |
| --- | --- |
| Existing Pulumi projects | Local source with `work_dir` |
| New embedded infrastructure | Inline source with `program` function |
| Different teams | Local source for independence |
| Packaged distribution | Inline source or bundled local |
| Multi-stack dependencies | Sequential deployment in order |
| Independent stacks | Parallel deployment with `ThreadPoolExecutor` |
| Create/select stack | `auto.create_or_select_stack(stack_name, project_name?, program?, work_dir?)` |
| Select existing stack | `auto.select_stack(stack_name, work_dir)` |
| Set config | `stack.set_config(key, auto.ConfigValue(value, secret?))` |
| Get outputs | `stack.outputs()` |
| Stream output | `stack.up(on_output=callback)` |
