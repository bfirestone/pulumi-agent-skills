# Python Examples: Pulumi Component Resource

## Component Anatomy

Full StaticSite component showing all four required elements:

```python
import pulumi
import pulumi_aws as aws


class StaticSiteArgs:
    def __init__(self,
                 index_document: pulumi.Input[str] = "index.html",
                 error_document: pulumi.Input[str] = "error.html"):
        self.index_document = index_document
        self.error_document = error_document


class StaticSite(pulumi.ComponentResource):
    bucket_name: pulumi.Output[str]
    website_url: pulumi.Output[str]

    def __init__(self, name: str, args: StaticSiteArgs,
                 opts: pulumi.ResourceOptions = None):
        # 1. Call super with type URN: <package>:<module>:<type>
        super().__init__("myorg:index:StaticSite", name, None, opts)

        # 2. Create child resources with parent=self
        bucket = aws.s3.Bucket(f"{name}-bucket",
            opts=pulumi.ResourceOptions(parent=self))

        website = aws.s3.BucketWebsiteConfigurationV2(f"{name}-website",
            bucket=bucket.id,
            index_document=aws.s3.BucketWebsiteConfigurationV2IndexDocumentArgs(
                suffix=args.index_document,
            ),
            error_document=aws.s3.BucketWebsiteConfigurationV2ErrorDocumentArgs(
                key=args.error_document,
            ),
            opts=pulumi.ResourceOptions(parent=self))

        # 3. Expose outputs as instance attributes
        self.bucket_name = bucket.id
        self.website_url = website.website_endpoint

        # 4. Register outputs -- always the last line
        self.register_outputs({
            "bucket_name": self.bucket_name,
            "website_url": self.website_url,
        })


# Usage
site = StaticSite("marketing", StaticSiteArgs())
pulumi.export("url", site.website_url)
```

In Python, extend `pulumi.ComponentResource` and call `super().__init__()` with the type URN. Accept `pulumi.ResourceOptions` as the opts parameter. Call `self.register_outputs()` as the last line of `__init__`.

## Registering Outputs Is Required

**Wrong** -- missing `register_outputs()`:

```python
class MyComponent(pulumi.ComponentResource):
    url: pulumi.Output[str]

    def __init__(self, name: str, args: MyArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:MyComponent", name, None, opts)
        bucket = aws.s3.Bucket(f"{name}-bucket",
            opts=pulumi.ResourceOptions(parent=self))
        self.url = bucket.bucket_regional_domain_name
        # Missing register_outputs -- component stuck "creating"
```

**Right**:

```python
class MyComponent(pulumi.ComponentResource):
    url: pulumi.Output[str]

    def __init__(self, name: str, args: MyArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:MyComponent", name, None, opts)
        bucket = aws.s3.Bucket(f"{name}-bucket",
            opts=pulumi.ResourceOptions(parent=self))
        self.url = bucket.bucket_regional_domain_name

        self.register_outputs({"url": self.url})
```

## Derive Child Names from the Component Name

**Wrong** -- hardcoded name:

```python
# Collides if two instances of this component exist
bucket = aws.s3.Bucket("my-bucket",
    opts=pulumi.ResourceOptions(parent=self))
```

**Right** -- derived from component name:

```python
# Unique per component instance
bucket = aws.s3.Bucket(f"{name}-bucket",
    opts=pulumi.ResourceOptions(parent=self))
```

## Wrap Properties in Input Types

In Python, use `pulumi.Input[T]` type annotations for args properties.

**Wrong**:

```python
class WebServiceArgs:
    def __init__(self,
                 port: int,          # Forces consumers to unwrap Outputs
                 vpc_id: str):       # Cannot accept vpc.id directly
        self.port = port
        self.vpc_id = vpc_id
```

**Right**:

```python
class WebServiceArgs:
    def __init__(self,
                 port: pulumi.Input[int],       # Accepts 8080 or some_output
                 vpc_id: pulumi.Input[str]):     # Accepts "vpc-123" or vpc.id
        self.port = port
        self.vpc_id = vpc_id
```

## Keep Structures Flat

```python
# Prefer flat
class DatabaseArgs:
    def __init__(self,
                 instance_class: pulumi.Input[str],
                 storage_gb: pulumi.Input[int],
                 enable_backups: pulumi.Input[bool] = None,
                 backup_retention_days: pulumi.Input[int] = None):
        self.instance_class = instance_class
        self.storage_gb = storage_gb
        self.enable_backups = enable_backups
        self.backup_retention_days = backup_retention_days

# Avoid deep nesting with nested dataclasses/dicts
```

## No Union Types

**Wrong**:

```python
class MyArgs:
    def __init__(self, port: pulumi.Input[str | int]):  # Fails in Go, C#
        self.port = port
```

**Right**:

```python
class MyArgs:
    def __init__(self, port: pulumi.Input[int]):  # Single type, works everywhere
        self.port = port
```

Use separate optional properties for variants:

```python
class StorageArgs:
    def __init__(self,
                 size_gb: pulumi.Input[int] = None,     # Specify size in GB
                 size_mb: pulumi.Input[int] = None):     # Or specify size in MB
        self.size_gb = size_gb
        self.size_mb = size_mb
```

## No Functions or Callbacks

**Wrong**:

```python
class MyArgs:
    def __init__(self, name_transform):  # Callable - cannot serialize
        self.name_transform = name_transform
```

**Right**:

```python
class MyArgs:
    def __init__(self,
                 name_prefix: pulumi.Input[str] = None,    # Configuration instead of callback
                 name_suffix: pulumi.Input[str] = None):
        self.name_prefix = name_prefix
        self.name_suffix = name_suffix
```

## Use Defaults for Optional Properties

Use `None` defaults and check with `is not None` in `__init__`:

```python
class SecureBucketArgs:
    def __init__(self,
                 enable_versioning: pulumi.Input[bool] = None,   # Defaults to True
                 enable_encryption: pulumi.Input[bool] = None,   # Defaults to True
                 block_public_access: pulumi.Input[bool] = None):  # Defaults to True
        self.enable_versioning = enable_versioning
        self.enable_encryption = enable_encryption
        self.block_public_access = block_public_access


class SecureBucket(pulumi.ComponentResource):
    def __init__(self, name: str, args: SecureBucketArgs = None,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:SecureBucket", name, None, opts)

        if args is None:
            args = SecureBucketArgs()

        enable_versioning = args.enable_versioning if args.enable_versioning is not None else True
        enable_encryption = args.enable_encryption if args.enable_encryption is not None else True
        block_public_access = args.block_public_access if args.block_public_access is not None else True

        # Apply defaults...


# Consumer only overrides what they need
bucket = SecureBucket("data", SecureBucketArgs(enable_versioning=False))
```

## Expose Only What Consumers Need

**Wrong** -- exposes implementation details:

```python
class Database(pulumi.ComponentResource):
    # Exposes everything -- consumers see implementation details
    cluster: aws.rds.Cluster
    primary_instance: aws.rds.ClusterInstance
    replica_instance: aws.rds.ClusterInstance
    subnet_group: aws.rds.SubnetGroup
    security_group: aws.ec2.SecurityGroup
    parameter_group: aws.rds.ClusterParameterGroup
    # ...
```

**Right** -- exposes only essentials:

```python
class Database(pulumi.ComponentResource):
    # Exposes only what consumers need
    endpoint: pulumi.Output[str]
    port: pulumi.Output[int]
    security_group_id: pulumi.Output[str]

    def __init__(self, name: str, args: DatabaseArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:Database", name, None, opts)

        sg = aws.ec2.SecurityGroup(f"{name}-sg", ...,
            opts=pulumi.ResourceOptions(parent=self))
        cluster = aws.rds.Cluster(f"{name}-cluster", ...,
            opts=pulumi.ResourceOptions(parent=self))

        self.endpoint = cluster.endpoint
        self.port = cluster.port
        self.security_group_id = sg.id

        self.register_outputs({
            "endpoint": self.endpoint,
            "port": self.port,
            "security_group_id": self.security_group_id,
        })
```

## Derive Composite Outputs

Use `Output.all` with `.apply()` to build derived values:

```python
self.connection_string = pulumi.Output.all(
    args.username, args.password, cluster.endpoint, cluster.port, args.database_name
).apply(lambda args: f"postgresql://{args[0]}:{args[1]}@{args[2]}:{args[3]}/{args[4]}")

self.register_outputs({"connection_string": self.connection_string})
```

## Sensible Defaults with Override

```python
class SecureBucketArgs:
    def __init__(self,
                 enable_versioning: pulumi.Input[bool] = None,
                 enable_encryption: pulumi.Input[bool] = None,
                 block_public_access: pulumi.Input[bool] = None,
                 tags: pulumi.Input[dict] = None):
        self.enable_versioning = enable_versioning
        self.enable_encryption = enable_encryption
        self.block_public_access = block_public_access
        self.tags = tags


class SecureBucket(pulumi.ComponentResource):
    bucket_id: pulumi.Output[str]
    arn: pulumi.Output[str]

    def __init__(self, name: str, args: SecureBucketArgs = None,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:SecureBucket", name, None, opts)

        if args is None:
            args = SecureBucketArgs()

        bucket = aws.s3.Bucket(f"{name}-bucket",
            tags=args.tags,
            opts=pulumi.ResourceOptions(parent=self))

        # Versioning on by default
        if args.enable_versioning is not False:
            aws.s3.BucketVersioningV2(f"{name}-versioning",
                bucket=bucket.id,
                versioning_configuration=aws.s3.BucketVersioningV2VersioningConfigurationArgs(
                    status="Enabled",
                ),
                opts=pulumi.ResourceOptions(parent=self))

        # Encryption on by default
        if args.enable_encryption is not False:
            aws.s3.BucketServerSideEncryptionConfigurationV2(f"{name}-encryption",
                bucket=bucket.id,
                rules=[aws.s3.BucketServerSideEncryptionConfigurationV2RuleArgs(
                    apply_server_side_encryption_by_default=aws.s3.BucketServerSideEncryptionConfigurationV2RuleApplyServerSideEncryptionByDefaultArgs(
                        sse_algorithm="AES256",
                    ),
                )],
                opts=pulumi.ResourceOptions(parent=self))

        # Public access blocked by default
        if args.block_public_access is not False:
            aws.s3.BucketPublicAccessBlock(f"{name}-public-access",
                bucket=bucket.id,
                block_public_acls=True,
                block_public_policy=True,
                ignore_public_acls=True,
                restrict_public_buckets=True,
                opts=pulumi.ResourceOptions(parent=self))

        self.bucket_id = bucket.id
        self.arn = bucket.arn
        self.register_outputs({"bucket_id": self.bucket_id, "arn": self.arn})
```

## Conditional Resource Creation

```python
class WebServiceArgs:
    def __init__(self,
                 image: pulumi.Input[str],
                 port: pulumi.Input[int],
                 enable_monitoring: pulumi.Input[bool] = None,
                 alarm_email: pulumi.Input[str] = None):
        self.image = image
        self.port = port
        self.enable_monitoring = enable_monitoring
        self.alarm_email = alarm_email


class WebService(pulumi.ComponentResource):
    def __init__(self, name: str, args: WebServiceArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:WebService", name, None, opts)

        service = aws.ecs.Service(f"{name}-service",
            # ...service config...
            opts=pulumi.ResourceOptions(parent=self))

        # Only create alarm infrastructure when monitoring is enabled
        if args.enable_monitoring:
            topic = aws.sns.Topic(f"{name}-alerts",
                opts=pulumi.ResourceOptions(parent=self))

            if args.alarm_email:
                aws.sns.TopicSubscription(f"{name}-alert-email",
                    topic=topic.arn,
                    protocol="email",
                    endpoint=args.alarm_email,
                    opts=pulumi.ResourceOptions(parent=self))

            aws.cloudwatch.MetricAlarm(f"{name}-cpu-alarm",
                # ...alarm config referencing service...
                alarm_actions=[topic.arn],
                opts=pulumi.ResourceOptions(parent=self))

        self.register_outputs({})
```

## Composition

```python
# Lower-level component
class VpcNetwork(pulumi.ComponentResource):
    vpc_id: pulumi.Output[str]
    public_subnet_ids: list[pulumi.Output[str]]
    private_subnet_ids: list[pulumi.Output[str]]

    def __init__(self, name: str, args: VpcNetworkArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:VpcNetwork", name, None, opts)
        # ...create VPC, subnets, route tables...
        self.register_outputs({"vpc_id": self.vpc_id})


# Higher-level component that uses VpcNetwork
class Platform(pulumi.ComponentResource):
    kubeconfig: pulumi.Output[str]

    def __init__(self, name: str, args: PlatformArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:index:Platform", name, None, opts)

        # Compose lower-level components
        network = VpcNetwork(f"{name}-network",
            VpcNetworkArgs(cidr_block=args.cidr_block),
            opts=pulumi.ResourceOptions(parent=self))

        cluster = aws.eks.Cluster(f"{name}-cluster",
            vpc_config=aws.eks.ClusterVpcConfigArgs(
                subnet_ids=network.private_subnet_ids,
            ),
            opts=pulumi.ResourceOptions(parent=self))

        self.kubeconfig = cluster.kubeconfig
        self.register_outputs({"kubeconfig": self.kubeconfig})
```

## Provider Passthrough

```python
# Consumer passes a provider for a different region
us_west = aws.Provider("us-west", region="us-west-2")
site = StaticSite("west-site", StaticSiteArgs(index_document="index.html"),
    opts=pulumi.ResourceOptions(providers=[us_west]))
```

## Entry Points

Entry point patterns for multi-language component providers in each authoring language.

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

**TypeScript** (`runtime: nodejs`):

Export component classes from `index.ts`. No separate entry point file is needed. Pulumi introspects exported classes automatically.

```typescript
// index.ts -- exports are the entry point
export { StaticSite, StaticSiteArgs } from "./staticSite";
export { SecureBucket, SecureBucketArgs } from "./secureBucket";
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
