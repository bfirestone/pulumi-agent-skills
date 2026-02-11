# TypeScript Examples: Pulumi Component Resource

## Component Anatomy

Full StaticSite component showing all four required elements:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

interface StaticSiteArgs {
    indexDocument?: pulumi.Input<string>;
    errorDocument?: pulumi.Input<string>;
}

class StaticSite extends pulumi.ComponentResource {
    public readonly bucketName: pulumi.Output<string>;
    public readonly websiteUrl: pulumi.Output<string>;

    constructor(name: string, args: StaticSiteArgs, opts?: pulumi.ComponentResourceOptions) {
        // 1. Call super with type URN: <package>:<module>:<type>
        super("myorg:index:StaticSite", name, {}, opts);

        // 2. Create child resources with parent: this
        const bucket = new aws.s3.Bucket(`${name}-bucket`, {}, { parent: this });

        const website = new aws.s3.BucketWebsiteConfigurationV2(`${name}-website`, {
            bucket: bucket.id,
            indexDocument: { suffix: args.indexDocument ?? "index.html" },
            errorDocument: { key: args.errorDocument ?? "error.html" },
        }, { parent: this });

        // 3. Expose outputs as class properties
        this.bucketName = bucket.id;
        this.websiteUrl = website.websiteEndpoint;

        // 4. Register outputs -- always the last line
        this.registerOutputs({
            bucketName: this.bucketName,
            websiteUrl: this.websiteUrl,
        });
    }
}

// Usage
const site = new StaticSite("marketing", {
    indexDocument: "index.html",
});
export const url = site.websiteUrl;
```

In TypeScript, extend `pulumi.ComponentResource` and call `super()` with the type URN. Accept `ComponentResourceOptions` as the third constructor parameter. Call `this.registerOutputs()` as the last line of the constructor.

## Registering Outputs Is Required

**Wrong** -- missing `registerOutputs()`:

```typescript
class MyComponent extends pulumi.ComponentResource {
    public readonly url: pulumi.Output<string>;

    constructor(name: string, args: MyArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:MyComponent", name, {}, opts);
        const bucket = new aws.s3.Bucket(`${name}-bucket`, {}, { parent: this });
        this.url = bucket.bucketRegionalDomainName;
        // Missing registerOutputs -- component stuck "creating"
    }
}
```

**Right**:

```typescript
class MyComponent extends pulumi.ComponentResource {
    public readonly url: pulumi.Output<string>;

    constructor(name: string, args: MyArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:MyComponent", name, {}, opts);
        const bucket = new aws.s3.Bucket(`${name}-bucket`, {}, { parent: this });
        this.url = bucket.bucketRegionalDomainName;

        this.registerOutputs({ url: this.url });
    }
}
```

## Derive Child Names from the Component Name

**Wrong** -- hardcoded name:

```typescript
// Collides if two instances of this component exist
const bucket = new aws.s3.Bucket("my-bucket", {}, { parent: this });
```

**Right** -- derived from component name:

```typescript
// Unique per component instance
const bucket = new aws.s3.Bucket(`${name}-bucket`, {}, { parent: this });
```

## Wrap Properties in Input Types

In TypeScript, use `pulumi.Input<T>` to wrap args properties.

**Wrong**:

```typescript
interface WebServiceArgs {
    port: number;            // Forces consumers to unwrap Outputs
    vpcId: string;           // Cannot accept vpc.id directly
}
```

**Right**:

```typescript
interface WebServiceArgs {
    port: pulumi.Input<number>;     // Accepts 8080 or someOutput
    vpcId: pulumi.Input<string>;    // Accepts "vpc-123" or vpc.id
}
```

## Keep Structures Flat

```typescript
// Prefer flat
interface DatabaseArgs {
    instanceClass: pulumi.Input<string>;
    storageGb: pulumi.Input<number>;
    enableBackups?: pulumi.Input<boolean>;
    backupRetentionDays?: pulumi.Input<number>;
}

// Avoid deep nesting
interface DatabaseArgs {
    instance: {
        compute: { class: pulumi.Input<string> };
        storage: { sizeGb: pulumi.Input<number> };
    };
    backup: {
        config: { enabled: pulumi.Input<boolean>; retention: pulumi.Input<number> };
    };
}
```

## No Union Types

**Wrong**:

```typescript
interface MyArgs {
    port: pulumi.Input<string | number>;  // Fails in Python, Go, C#
}
```

**Right**:

```typescript
interface MyArgs {
    port: pulumi.Input<number>;  // Single type, works everywhere
}
```

Use separate optional properties for variants:

```typescript
interface StorageArgs {
    sizeGb?: pulumi.Input<number>;      // Specify size in GB
    sizeMb?: pulumi.Input<number>;      // Or specify size in MB
}
```

## No Functions or Callbacks

**Wrong**:

```typescript
interface MyArgs {
    nameTransform: (name: string) => string;  // Cannot serialize
}
```

**Right**:

```typescript
interface MyArgs {
    namePrefix?: pulumi.Input<string>;   // Configuration instead of callback
    nameSuffix?: pulumi.Input<string>;
}
```

## Use Defaults for Optional Properties

Use the `??` operator to apply defaults in the constructor:

```typescript
interface SecureBucketArgs {
    enableVersioning?: pulumi.Input<boolean>;   // Defaults to true
    enableEncryption?: pulumi.Input<boolean>;   // Defaults to true
    blockPublicAccess?: pulumi.Input<boolean>;  // Defaults to true
}

class SecureBucket extends pulumi.ComponentResource {
    constructor(name: string, args: SecureBucketArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:SecureBucket", name, {}, opts);

        const enableVersioning = args.enableVersioning ?? true;
        const enableEncryption = args.enableEncryption ?? true;
        const blockPublicAccess = args.blockPublicAccess ?? true;

        // Apply defaults...
    }
}

// Consumer only overrides what they need
const bucket = new SecureBucket("data", { enableVersioning: false });
```

## Expose Only What Consumers Need

**Wrong** -- exposes implementation details:

```typescript
class Database extends pulumi.ComponentResource {
    // Exposes everything -- consumers see implementation details
    public readonly cluster: aws.rds.Cluster;
    public readonly primaryInstance: aws.rds.ClusterInstance;
    public readonly replicaInstance: aws.rds.ClusterInstance;
    public readonly subnetGroup: aws.rds.SubnetGroup;
    public readonly securityGroup: aws.ec2.SecurityGroup;
    public readonly parameterGroup: aws.rds.ClusterParameterGroup;
    // ...
}
```

**Right** -- exposes only essentials:

```typescript
class Database extends pulumi.ComponentResource {
    // Exposes only what consumers need
    public readonly endpoint: pulumi.Output<string>;
    public readonly port: pulumi.Output<number>;
    public readonly securityGroupId: pulumi.Output<string>;

    constructor(name: string, args: DatabaseArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:Database", name, {}, opts);

        const sg = new aws.ec2.SecurityGroup(`${name}-sg`, { /* ... */ }, { parent: this });
        const cluster = new aws.rds.Cluster(`${name}-cluster`, { /* ... */ }, { parent: this });

        this.endpoint = cluster.endpoint;
        this.port = cluster.port;
        this.securityGroupId = sg.id;

        this.registerOutputs({
            endpoint: this.endpoint,
            port: this.port,
            securityGroupId: this.securityGroupId,
        });
    }
}
```

## Derive Composite Outputs

Use `pulumi.interpolate` or `pulumi.concat` to build derived values:

```typescript
this.connectionString = pulumi.interpolate`postgresql://${args.username}:${args.password}@${cluster.endpoint}:${cluster.port}/${args.databaseName}`;

this.registerOutputs({ connectionString: this.connectionString });
```

## Sensible Defaults with Override

```typescript
interface SecureBucketArgs {
    enableVersioning?: pulumi.Input<boolean>;
    enableEncryption?: pulumi.Input<boolean>;
    blockPublicAccess?: pulumi.Input<boolean>;
    tags?: pulumi.Input<Record<string, pulumi.Input<string>>>;
}

class SecureBucket extends pulumi.ComponentResource {
    public readonly bucketId: pulumi.Output<string>;
    public readonly arn: pulumi.Output<string>;

    constructor(name: string, args: SecureBucketArgs = {}, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:SecureBucket", name, {}, opts);

        const bucket = new aws.s3.Bucket(`${name}-bucket`, {
            tags: args.tags,
        }, { parent: this });

        // Versioning on by default
        if (args.enableVersioning !== false) {
            new aws.s3.BucketVersioningV2(`${name}-versioning`, {
                bucket: bucket.id,
                versioningConfiguration: { status: "Enabled" },
            }, { parent: this });
        }

        // Encryption on by default
        if (args.enableEncryption !== false) {
            new aws.s3.BucketServerSideEncryptionConfigurationV2(`${name}-encryption`, {
                bucket: bucket.id,
                rules: [{ applyServerSideEncryptionByDefault: { sseAlgorithm: "AES256" } }],
            }, { parent: this });
        }

        // Public access blocked by default
        if (args.blockPublicAccess !== false) {
            new aws.s3.BucketPublicAccessBlock(`${name}-public-access`, {
                bucket: bucket.id,
                blockPublicAcls: true,
                blockPublicPolicy: true,
                ignorePublicAcls: true,
                restrictPublicBuckets: true,
            }, { parent: this });
        }

        this.bucketId = bucket.id;
        this.arn = bucket.arn;
        this.registerOutputs({ bucketId: this.bucketId, arn: this.arn });
    }
}
```

## Conditional Resource Creation

```typescript
interface WebServiceArgs {
    image: pulumi.Input<string>;
    port: pulumi.Input<number>;
    enableMonitoring?: pulumi.Input<boolean>;
    alarmEmail?: pulumi.Input<string>;
}

class WebService extends pulumi.ComponentResource {
    constructor(name: string, args: WebServiceArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:WebService", name, {}, opts);

        const service = new aws.ecs.Service(`${name}-service`, {
            // ...service config...
        }, { parent: this });

        // Only create alarm infrastructure when monitoring is enabled
        if (args.enableMonitoring) {
            const topic = new aws.sns.Topic(`${name}-alerts`, {}, { parent: this });

            if (args.alarmEmail) {
                new aws.sns.TopicSubscription(`${name}-alert-email`, {
                    topic: topic.arn,
                    protocol: "email",
                    endpoint: args.alarmEmail,
                }, { parent: this });
            }

            new aws.cloudwatch.MetricAlarm(`${name}-cpu-alarm`, {
                // ...alarm config referencing service...
                alarmActions: [topic.arn],
            }, { parent: this });
        }

        this.registerOutputs({});
    }
}
```

## Composition

```typescript
// Lower-level component
class VpcNetwork extends pulumi.ComponentResource {
    public readonly vpcId: pulumi.Output<string>;
    public readonly publicSubnetIds: pulumi.Output<string>[];
    public readonly privateSubnetIds: pulumi.Output<string>[];

    constructor(name: string, args: VpcNetworkArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:VpcNetwork", name, {}, opts);
        // ...create VPC, subnets, route tables...
        this.registerOutputs({ vpcId: this.vpcId });
    }
}

// Higher-level component that uses VpcNetwork
class Platform extends pulumi.ComponentResource {
    public readonly kubeconfig: pulumi.Output<string>;

    constructor(name: string, args: PlatformArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:index:Platform", name, {}, opts);

        // Compose lower-level components
        const network = new VpcNetwork(`${name}-network`, {
            cidrBlock: args.cidrBlock,
        }, { parent: this });

        const cluster = new aws.eks.Cluster(`${name}-cluster`, {
            vpcConfig: {
                subnetIds: network.privateSubnetIds,
            },
        }, { parent: this });

        this.kubeconfig = cluster.kubeconfig;
        this.registerOutputs({ kubeconfig: this.kubeconfig });
    }
}
```

## Provider Passthrough

```typescript
// Consumer passes a provider for a different region
const usWest = new aws.Provider("us-west", { region: "us-west-2" });
const site = new StaticSite("west-site", { indexDocument: "index.html" }, {
    providers: [usWest],
});
```

## Entry Points

Entry point patterns for multi-language component providers in each authoring language.

**TypeScript** (`runtime: nodejs`):

Export component classes from `index.ts`. No separate entry point file is needed. Pulumi introspects exported classes automatically.

```typescript
// index.ts -- exports are the entry point
export { StaticSite, StaticSiteArgs } from "./staticSite";
export { SecureBucket, SecureBucketArgs } from "./secureBucket";
```

**Python** (`runtime: python`):

Create a `__main__.py` that calls `component_provider_host` with all component classes:

```python
from pulumi.provider.experimental import component_provider_host
from static_site import StaticSite
from secure_bucket import SecureBucket

if __name__ == "__main__":
    component_provider_host(
        name="my-components",
        components=[StaticSite, SecureBucket],
    )
```

**Go** (`runtime: go`):

Create a `main.go` that builds and runs the provider:

```go
package main

import (
    "context"
    "fmt"
    "os"

    "github.com/pulumi/pulumi-go-provider/infer"
)

func main() {
    p, err := infer.NewProviderBuilder().
        WithComponents(
            infer.ComponentF(NewStaticSite),
            infer.ComponentF(NewSecureBucket),
        ).
        Build()
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    if err := p.Run(context.Background(), "my-components", "0.1.0"); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

**C#** (`runtime: dotnet`):

Create a `Program.cs` that serves the component provider host:

```csharp
using System.Threading.Tasks;

class Program
{
    public static Task Main(string[] args) =>
        Pulumi.Experimental.Provider.ComponentProviderHost.Serve(args);
}
```
