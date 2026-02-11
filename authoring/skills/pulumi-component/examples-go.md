# Go Examples: Pulumi Component Resource

## Component Anatomy

Full StaticSite component showing all four required elements:

```go
package main

import (
	"github.com/pulumi/pulumi-aws/sdk/v7/go/aws/s3"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

type StaticSiteArgs struct {
	IndexDocument pulumi.StringInput `pulumi:"indexDocument"`
	ErrorDocument pulumi.StringInput `pulumi:"errorDocument"`
}

type StaticSite struct {
	pulumi.ResourceState
	BucketName pulumi.StringOutput `pulumi:"bucketName"`
	WebsiteUrl pulumi.StringOutput `pulumi:"websiteUrl"`
}

func NewStaticSite(ctx *pulumi.Context, name string, args *StaticSiteArgs, opts ...pulumi.ResourceOption) (*StaticSite, error) {
	// 1. Register the component with type URN: <package>:<module>:<type>
	component := &StaticSite{}
	err := ctx.RegisterComponentResource("myorg:index:StaticSite", name, component, opts...)
	if err != nil {
		return nil, err
	}

	// 2. Create child resources with pulumi.Parent(component)
	bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{},
		pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	indexDoc := args.IndexDocument
	if indexDoc == nil {
		indexDoc = pulumi.String("index.html")
	}
	errorDoc := args.ErrorDocument
	if errorDoc == nil {
		errorDoc = pulumi.String("error.html")
	}

	website, err := s3.NewBucketWebsiteConfigurationV2(ctx, name+"-website", &s3.BucketWebsiteConfigurationV2Args{
		Bucket: bucket.ID(),
		IndexDocument: &s3.BucketWebsiteConfigurationV2IndexDocumentArgs{
			Suffix: indexDoc,
		},
		ErrorDocument: &s3.BucketWebsiteConfigurationV2ErrorDocumentArgs{
			Key: errorDoc,
		},
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	// 3. Expose outputs as struct fields
	component.BucketName = bucket.ID()
	component.WebsiteUrl = website.WebsiteEndpoint

	// 4. Register outputs -- always the last step
	ctx.RegisterResourceOutputs(component, pulumi.Map{
		"bucketName": component.BucketName,
		"websiteUrl": component.WebsiteUrl,
	})

	return component, nil
}

// Usage
func main() {
	pulumi.Run(func(ctx *pulumi.Context) error {
		site, err := NewStaticSite(ctx, "marketing", &StaticSiteArgs{
			IndexDocument: pulumi.String("index.html"),
		})
		if err != nil {
			return err
		}
		ctx.Export("url", site.WebsiteUrl)
		return nil
	})
}
```

In Go, embed `pulumi.ResourceState` in a struct and call `ctx.RegisterComponentResource()` with the type URN. Accept `pulumi.ResourceOption` variadic opts. Call `ctx.RegisterResourceOutputs()` as the last step.

## Registering Outputs Is Required

**Wrong** -- missing `ctx.RegisterResourceOutputs()`:

```go
func NewMyComponent(ctx *pulumi.Context, name string, args *MyArgs, opts ...pulumi.ResourceOption) (*MyComponent, error) {
	component := &MyComponent{}
	err := ctx.RegisterComponentResource("myorg:index:MyComponent", name, component, opts...)
	if err != nil {
		return nil, err
	}
	bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}
	component.Url = bucket.BucketRegionalDomainName
	// Missing RegisterResourceOutputs -- component stuck "creating"
	return component, nil
}
```

**Right**:

```go
func NewMyComponent(ctx *pulumi.Context, name string, args *MyArgs, opts ...pulumi.ResourceOption) (*MyComponent, error) {
	component := &MyComponent{}
	err := ctx.RegisterComponentResource("myorg:index:MyComponent", name, component, opts...)
	if err != nil {
		return nil, err
	}
	bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}
	component.Url = bucket.BucketRegionalDomainName

	ctx.RegisterResourceOutputs(component, pulumi.Map{"url": component.Url})
	return component, nil
}
```

## Derive Child Names from the Component Name

**Wrong** -- hardcoded name:

```go
// Collides if two instances of this component exist
bucket, err := s3.NewBucket(ctx, "my-bucket", &s3.BucketArgs{}, pulumi.Parent(component))
```

**Right** -- derived from component name:

```go
// Unique per component instance
bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{}, pulumi.Parent(component))
```

## Wrap Properties in Input Types

In Go, use `pulumi.XxxInput` types (e.g., `pulumi.StringInput`, `pulumi.IntInput`) and add `pulumi` struct tags.

**Wrong**:

```go
type WebServiceArgs struct {
	Port  int    // Forces consumers to unwrap Outputs
	VpcId string // Cannot accept vpc.ID() directly
}
```

**Right**:

```go
type WebServiceArgs struct {
	Port  pulumi.IntInput    `pulumi:"port"`    // Accepts 8080 or someOutput
	VpcId pulumi.StringInput `pulumi:"vpcId"`   // Accepts "vpc-123" or vpc.ID()
}
```

## Keep Structures Flat

```go
// Prefer flat
type DatabaseArgs struct {
	InstanceClass      pulumi.StringInput `pulumi:"instanceClass"`
	StorageGb          pulumi.IntInput    `pulumi:"storageGb"`
	EnableBackups      pulumi.BoolInput   `pulumi:"enableBackups"`
	BackupRetentionDays pulumi.IntInput   `pulumi:"backupRetentionDays"`
}

// Avoid deep nesting
type DatabaseArgs struct {
	Instance struct {
		Compute struct { Class pulumi.StringInput }
		Storage struct { SizeGb pulumi.IntInput }
	}
	Backup struct {
		Config struct { Enabled pulumi.BoolInput; Retention pulumi.IntInput }
	}
}
```

## No Union Types

**Wrong**:

```go
// Go doesn't have union types, but avoid interface{} as a workaround
type MyArgs struct {
	Port interface{} // Fails in Python, C#, Java
}
```

**Right**:

```go
type MyArgs struct {
	Port pulumi.IntInput `pulumi:"port"` // Single type, works everywhere
}
```

Use separate optional properties for variants:

```go
type StorageArgs struct {
	SizeGb pulumi.IntInput `pulumi:"sizeGb"` // Specify size in GB
	SizeMb pulumi.IntInput `pulumi:"sizeMb"` // Or specify size in MB
}
```

## No Functions or Callbacks

**Wrong**:

```go
type MyArgs struct {
	NameTransform func(string) string // Cannot serialize
}
```

**Right**:

```go
type MyArgs struct {
	NamePrefix pulumi.StringInput `pulumi:"namePrefix"` // Configuration instead of callback
	NameSuffix pulumi.StringInput `pulumi:"nameSuffix"`
}
```

## Use Defaults for Optional Properties

Use `*bool` for optional boolean fields and nil-check in the constructor:

```go
type SecureBucketArgs struct {
	EnableVersioning  *bool `pulumi:"enableVersioning"`   // Defaults to true
	EnableEncryption  *bool `pulumi:"enableEncryption"`   // Defaults to true
	BlockPublicAccess *bool `pulumi:"blockPublicAccess"`  // Defaults to true
}

func NewSecureBucket(ctx *pulumi.Context, name string, args *SecureBucketArgs, opts ...pulumi.ResourceOption) (*SecureBucket, error) {
	component := &SecureBucket{}
	err := ctx.RegisterComponentResource("myorg:index:SecureBucket", name, component, opts...)
	if err != nil {
		return nil, err
	}

	enableVersioning := true
	if args != nil && args.EnableVersioning != nil {
		enableVersioning = *args.EnableVersioning
	}
	enableEncryption := true
	if args != nil && args.EnableEncryption != nil {
		enableEncryption = *args.EnableEncryption
	}
	blockPublicAccess := true
	if args != nil && args.BlockPublicAccess != nil {
		blockPublicAccess = *args.BlockPublicAccess
	}

	// Apply defaults...
	return component, nil
}

// Consumer only overrides what they need
f := false
bucket, err := NewSecureBucket(ctx, "data", &SecureBucketArgs{EnableVersioning: &f})
```

## Expose Only What Consumers Need

**Wrong** -- exposes implementation details:

```go
type Database struct {
	pulumi.ResourceState
	// Exposes everything -- consumers see implementation details
	Cluster          *rds.Cluster
	PrimaryInstance  *rds.ClusterInstance
	ReplicaInstance  *rds.ClusterInstance
	SubnetGroup      *rds.SubnetGroup
	SecurityGroup    *ec2.SecurityGroup
	ParameterGroup   *rds.ClusterParameterGroup
}
```

**Right** -- exposes only essentials:

```go
type Database struct {
	pulumi.ResourceState
	// Exposes only what consumers need
	Endpoint        pulumi.StringOutput `pulumi:"endpoint"`
	Port            pulumi.IntOutput    `pulumi:"port"`
	SecurityGroupId pulumi.StringOutput `pulumi:"securityGroupId"`
}

func NewDatabase(ctx *pulumi.Context, name string, args *DatabaseArgs, opts ...pulumi.ResourceOption) (*Database, error) {
	component := &Database{}
	err := ctx.RegisterComponentResource("myorg:index:Database", name, component, opts...)
	if err != nil {
		return nil, err
	}

	sg, err := ec2.NewSecurityGroup(ctx, name+"-sg", &ec2.SecurityGroupArgs{ /* ... */ },
		pulumi.Parent(component))
	if err != nil {
		return nil, err
	}
	cluster, err := rds.NewCluster(ctx, name+"-cluster", &rds.ClusterArgs{ /* ... */ },
		pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	component.Endpoint = cluster.Endpoint
	component.Port = cluster.Port
	component.SecurityGroupId = sg.ID()

	ctx.RegisterResourceOutputs(component, pulumi.Map{
		"endpoint":        component.Endpoint,
		"port":            component.Port,
		"securityGroupId": component.SecurityGroupId,
	})
	return component, nil
}
```

## Derive Composite Outputs

Use `pulumi.Sprintf` to build derived values:

```go
component.ConnectionString = pulumi.Sprintf(
	"postgresql://%s:%s@%s:%v/%s",
	args.Username, args.Password, cluster.Endpoint, cluster.Port, args.DatabaseName,
)

ctx.RegisterResourceOutputs(component, pulumi.Map{
	"connectionString": component.ConnectionString,
})
```

## Sensible Defaults with Override

```go
type SecureBucketArgs struct {
	EnableVersioning  *bool                  `pulumi:"enableVersioning"`
	EnableEncryption  *bool                  `pulumi:"enableEncryption"`
	BlockPublicAccess *bool                  `pulumi:"blockPublicAccess"`
	Tags              pulumi.StringMapInput  `pulumi:"tags"`
}

type SecureBucket struct {
	pulumi.ResourceState
	BucketId pulumi.StringOutput `pulumi:"bucketId"`
	Arn      pulumi.StringOutput `pulumi:"arn"`
}

func NewSecureBucket(ctx *pulumi.Context, name string, args *SecureBucketArgs, opts ...pulumi.ResourceOption) (*SecureBucket, error) {
	component := &SecureBucket{}
	err := ctx.RegisterComponentResource("myorg:index:SecureBucket", name, component, opts...)
	if err != nil {
		return nil, err
	}

	bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{
		Tags: args.Tags,
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	// Versioning on by default
	enableVersioning := true
	if args != nil && args.EnableVersioning != nil {
		enableVersioning = *args.EnableVersioning
	}
	if enableVersioning {
		_, err = s3.NewBucketVersioningV2(ctx, name+"-versioning", &s3.BucketVersioningV2Args{
			Bucket: bucket.ID(),
			VersioningConfiguration: &s3.BucketVersioningV2VersioningConfigurationArgs{
				Status: pulumi.String("Enabled"),
			},
		}, pulumi.Parent(component))
		if err != nil {
			return nil, err
		}
	}

	// Encryption on by default
	enableEncryption := true
	if args != nil && args.EnableEncryption != nil {
		enableEncryption = *args.EnableEncryption
	}
	if enableEncryption {
		_, err = s3.NewBucketServerSideEncryptionConfigurationV2(ctx, name+"-encryption", &s3.BucketServerSideEncryptionConfigurationV2Args{
			Bucket: bucket.ID(),
			Rules: s3.BucketServerSideEncryptionConfigurationV2RuleArray{
				&s3.BucketServerSideEncryptionConfigurationV2RuleArgs{
					ApplyServerSideEncryptionByDefault: &s3.BucketServerSideEncryptionConfigurationV2RuleApplyServerSideEncryptionByDefaultArgs{
						SseAlgorithm: pulumi.String("AES256"),
					},
				},
			},
		}, pulumi.Parent(component))
		if err != nil {
			return nil, err
		}
	}

	// Public access blocked by default
	blockPublicAccess := true
	if args != nil && args.BlockPublicAccess != nil {
		blockPublicAccess = *args.BlockPublicAccess
	}
	if blockPublicAccess {
		_, err = s3.NewBucketPublicAccessBlock(ctx, name+"-public-access", &s3.BucketPublicAccessBlockArgs{
			Bucket:                bucket.ID(),
			BlockPublicAcls:       pulumi.Bool(true),
			BlockPublicPolicy:     pulumi.Bool(true),
			IgnorePublicAcls:      pulumi.Bool(true),
			RestrictPublicBuckets: pulumi.Bool(true),
		}, pulumi.Parent(component))
		if err != nil {
			return nil, err
		}
	}

	component.BucketId = bucket.ID()
	component.Arn = bucket.Arn
	ctx.RegisterResourceOutputs(component, pulumi.Map{
		"bucketId": component.BucketId,
		"arn":      component.Arn,
	})
	return component, nil
}
```

## Conditional Resource Creation

```go
type WebServiceArgs struct {
	Image            pulumi.StringInput `pulumi:"image"`
	Port             pulumi.IntInput    `pulumi:"port"`
	EnableMonitoring *bool              `pulumi:"enableMonitoring"`
	AlarmEmail       pulumi.StringInput `pulumi:"alarmEmail"`
}

func NewWebService(ctx *pulumi.Context, name string, args *WebServiceArgs, opts ...pulumi.ResourceOption) (*WebService, error) {
	component := &WebService{}
	err := ctx.RegisterComponentResource("myorg:index:WebService", name, component, opts...)
	if err != nil {
		return nil, err
	}

	_, err = ecs.NewService(ctx, name+"-service", &ecs.ServiceArgs{
		// ...service config...
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	// Only create alarm infrastructure when monitoring is enabled
	if args.EnableMonitoring != nil && *args.EnableMonitoring {
		topic, err := sns.NewTopic(ctx, name+"-alerts", &sns.TopicArgs{},
			pulumi.Parent(component))
		if err != nil {
			return nil, err
		}

		if args.AlarmEmail != nil {
			_, err = sns.NewTopicSubscription(ctx, name+"-alert-email", &sns.TopicSubscriptionArgs{
				Topic:    topic.Arn,
				Protocol: pulumi.String("email"),
				Endpoint: args.AlarmEmail,
			}, pulumi.Parent(component))
			if err != nil {
				return nil, err
			}
		}

		_, err = cloudwatch.NewMetricAlarm(ctx, name+"-cpu-alarm", &cloudwatch.MetricAlarmArgs{
			// ...alarm config referencing service...
			AlarmActions: pulumi.StringArray{topic.Arn},
		}, pulumi.Parent(component))
		if err != nil {
			return nil, err
		}
	}

	ctx.RegisterResourceOutputs(component, pulumi.Map{})
	return component, nil
}
```

## Composition

```go
// Lower-level component
type VpcNetwork struct {
	pulumi.ResourceState
	VpcId            pulumi.StringOutput   `pulumi:"vpcId"`
	PublicSubnetIds  pulumi.StringArrayOutput `pulumi:"publicSubnetIds"`
	PrivateSubnetIds pulumi.StringArrayOutput `pulumi:"privateSubnetIds"`
}

func NewVpcNetwork(ctx *pulumi.Context, name string, args *VpcNetworkArgs, opts ...pulumi.ResourceOption) (*VpcNetwork, error) {
	component := &VpcNetwork{}
	err := ctx.RegisterComponentResource("myorg:index:VpcNetwork", name, component, opts...)
	if err != nil {
		return nil, err
	}
	// ...create VPC, subnets, route tables...
	ctx.RegisterResourceOutputs(component, pulumi.Map{"vpcId": component.VpcId})
	return component, nil
}

// Higher-level component that uses VpcNetwork
type Platform struct {
	pulumi.ResourceState
	Kubeconfig pulumi.StringOutput `pulumi:"kubeconfig"`
}

func NewPlatform(ctx *pulumi.Context, name string, args *PlatformArgs, opts ...pulumi.ResourceOption) (*Platform, error) {
	component := &Platform{}
	err := ctx.RegisterComponentResource("myorg:index:Platform", name, component, opts...)
	if err != nil {
		return nil, err
	}

	// Compose lower-level components
	network, err := NewVpcNetwork(ctx, name+"-network", &VpcNetworkArgs{
		CidrBlock: args.CidrBlock,
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	cluster, err := eks.NewCluster(ctx, name+"-cluster", &eks.ClusterArgs{
		VpcConfig: &eks.ClusterVpcConfigArgs{
			SubnetIds: network.PrivateSubnetIds,
		},
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	component.Kubeconfig = cluster.Kubeconfig
	ctx.RegisterResourceOutputs(component, pulumi.Map{"kubeconfig": component.Kubeconfig})
	return component, nil
}
```

## Provider Passthrough

```go
// Consumer passes a provider for a different region
usWest, err := aws.NewProvider(ctx, "us-west", &aws.ProviderArgs{
	Region: pulumi.String("us-west-2"),
})
if err != nil {
	return err
}
site, err := NewStaticSite(ctx, "west-site", &StaticSiteArgs{
	IndexDocument: pulumi.String("index.html"),
}, pulumi.Providers(usWest))
```

## Entry Points

Entry point patterns for multi-language component providers in each authoring language.

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
