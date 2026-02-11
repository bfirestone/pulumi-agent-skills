# Go Examples: Pulumi Automation API

## What is Automation API

Basic usage — create or select a stack and run `Up`:

```go
package main

import (
	"context"
	"fmt"

	"github.com/pulumi/pulumi/sdk/v3/go/auto"
	"github.com/pulumi/pulumi/sdk/v3/go/auto/optup"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
	ctx := context.Background()

	// Create or select a stack
	s, err := auto.UpsertStackInlineSource(ctx, "dev", "my-project",
		func(ctx *pulumi.Context) error {
			// Your Pulumi program here
			return nil
		})
	if err != nil {
		panic(err)
	}

	// Run pulumi up programmatically
	res, err := s.Up(ctx, optup.ProgressStreams(fmt.Stdout))
	if err != nil {
		panic(err)
	}
	fmt.Printf("Update summary: %v\n", res.Summary)
}
```

## Architecture Choices

### Local Source

```go
ctx := context.Background()

s, err := auto.UpsertStackLocalSource(ctx, "dev", "./infrastructure")
if err != nil {
	panic(err)
}
```

### Inline Source

```go
ctx := context.Background()

s, err := auto.UpsertStackInlineSource(ctx, "dev", "my-project",
	func(ctx *pulumi.Context) error {
		_, err := s3.NewBucket(ctx, "my-bucket", &s3.BucketArgs{})
		if err != nil {
			return err
		}
		return nil
	})
if err != nil {
	panic(err)
}
```

## Common Patterns

### Multi-Stack Orchestration

Deploy multiple stacks in dependency order:

```go
package main

import (
	"context"
	"fmt"

	"github.com/pulumi/pulumi/sdk/v3/go/auto"
	"github.com/pulumi/pulumi/sdk/v3/go/auto/optup"
	"github.com/pulumi/pulumi/sdk/v3/go/auto/optdestroy"
)

func deploy() error {
	ctx := context.Background()

	stacks := []struct {
		name string
		dir  string
	}{
		{"infrastructure", "./infra"},
		{"platform", "./platform"},
		{"application", "./app"},
	}

	for _, stackInfo := range stacks {
		fmt.Printf("Deploying %s...\n", stackInfo.name)

		s, err := auto.UpsertStackLocalSource(ctx, "prod", stackInfo.dir)
		if err != nil {
			return err
		}

		_, err = s.Up(ctx, optup.ProgressStreams(fmt.Stdout))
		if err != nil {
			return err
		}
		fmt.Printf("%s deployed successfully\n", stackInfo.name)
	}
	return nil
}

func destroy() error {
	ctx := context.Background()

	// Destroy in reverse order
	stacks := []struct {
		name string
		dir  string
	}{
		{"application", "./app"},
		{"platform", "./platform"},
		{"infrastructure", "./infra"},
	}

	for _, stackInfo := range stacks {
		fmt.Printf("Destroying %s...\n", stackInfo.name)

		s, err := auto.SelectStackLocalSource(ctx, "prod", stackInfo.dir)
		if err != nil {
			return err
		}

		_, err = s.Destroy(ctx, optdestroy.ProgressStreams(fmt.Stdout))
		if err != nil {
			return err
		}
	}
	return nil
}
```

### Passing Configuration

```go
ctx := context.Background()

s, err := auto.UpsertStackLocalSource(ctx, "dev", "./infrastructure")
if err != nil {
	return err
}

// Set configuration values
err = s.SetConfig(ctx, "aws:region", auto.ConfigValue{Value: "us-west-2"})
if err != nil {
	return err
}
err = s.SetConfig(ctx, "dbPassword", auto.ConfigValue{Value: "secret", Secret: true})
if err != nil {
	return err
}

// Then deploy
_, err = s.Up(ctx)
```

### Reading Outputs

```go
res, err := s.Up(ctx)
if err != nil {
	return err
}

// Get all outputs
outputs, err := s.Outputs(ctx)
if err != nil {
	return err
}
fmt.Printf("VPC ID: %s\n", outputs["vpcId"].Value)

// Or from the up result
fmt.Printf("Outputs: %v\n", res.Outputs)
```

### Error Handling

```go
res, err := s.Up(ctx, optup.ProgressStreams(fmt.Stdout))
if err != nil {
	fmt.Printf("Deployment error: %v\n", err)
	return err
}

if res.Summary.Result == "failed" {
	fmt.Println("Deployment failed")
	os.Exit(1)
}
```

### Parallel Stack Operations

```go
independentStacks := []struct {
	name string
	dir  string
}{
	{"service-a", "./service-a"},
	{"service-b", "./service-b"},
	{"service-c", "./service-c"},
}

var wg sync.WaitGroup
errCh := make(chan error, len(independentStacks))

for _, stackInfo := range independentStacks {
	wg.Add(1)
	go func(info struct{ name, dir string }) {
		defer wg.Done()
		s, err := auto.UpsertStackLocalSource(ctx, "prod", info.dir)
		if err != nil {
			errCh <- fmt.Errorf("[%s] %w", info.name, err)
			return
		}
		_, err = s.Up(ctx, optup.ProgressStreams(os.Stdout))
		if err != nil {
			errCh <- fmt.Errorf("[%s] %w", info.name, err)
		}
	}(stackInfo)
}

wg.Wait()
close(errCh)

for err := range errCh {
	fmt.Printf("Error: %v\n", err)
}
```

## Best Practices

### Separate Configuration from Code

```go
import (
	"encoding/json"
	"os"
)

type DeployConfig struct {
	Stacks []struct {
		Name string `json:"name"`
		Dir  string `json:"dir"`
	} `json:"stacks"`
	Environment string `json:"environment"`
}

func loadConfig() (*DeployConfig, error) {
	data, err := os.ReadFile("./deploy-config.json")
	if err != nil {
		return nil, err
	}
	var cfg DeployConfig
	err = json.Unmarshal(data, &cfg)
	return &cfg, err
}

func deploy() error {
	ctx := context.Background()
	cfg, err := loadConfig()
	if err != nil {
		return err
	}

	for _, stackInfo := range cfg.Stacks {
		s, err := auto.UpsertStackLocalSource(ctx, cfg.Environment, stackInfo.Dir)
		if err != nil {
			return err
		}
		_, err = s.Up(ctx)
		if err != nil {
			return err
		}
	}
	return nil
}
```

### Stream Output for Long Operations

```go
_, err = s.Up(ctx, optup.ProgressStreams(os.Stdout))
// Or send to a custom writer, websocket, logging system, etc.
```

## Quick Reference

| Scenario | Approach |
| --- | --- |
| Existing Pulumi projects | Local source with `auto.UpsertStackLocalSource()` |
| New embedded infrastructure | Inline source with `auto.UpsertStackInlineSource()` |
| Different teams | Local source for independence |
| Compiled binary distribution | Inline source or bundled local |
| Multi-stack dependencies | Sequential deployment in order |
| Independent stacks | Parallel deployment with goroutines + `sync.WaitGroup` |
| Create/select stack (local) | `auto.UpsertStackLocalSource(ctx, stackName, workDir)` |
| Create/select stack (inline) | `auto.UpsertStackInlineSource(ctx, stackName, projectName, program)` |
| Select existing stack | `auto.SelectStackLocalSource(ctx, stackName, workDir)` |
| Set config | `s.SetConfig(ctx, key, auto.ConfigValue{Value, Secret})` |
| Get outputs | `s.Outputs(ctx)` |
| Stream output | `optup.ProgressStreams(os.Stdout)` |
