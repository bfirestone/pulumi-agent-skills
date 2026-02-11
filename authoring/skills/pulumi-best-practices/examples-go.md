# Go Examples: Pulumi Best Practices

## Practice 1: Never Create Resources Inside `ApplyT()`

Wrong approach — resource won't appear in preview:

```go
package main

import (
	"github.com/pulumi/pulumi-aws/sdk/v7/go/aws/s3"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
	pulumi.Run(func(ctx *pulumi.Context) error {
		bucket, err := s3.NewBucket(ctx, "bucket", &s3.BucketArgs{})
		if err != nil {
			return err
		}

		// WRONG: This resource won't appear in preview
		bucket.ID().ApplyT(func(bucketId string) error {
			_, err := s3.NewBucketObject(ctx, "object", &s3.BucketObjectArgs{
				Bucket:  pulumi.String(bucketId),
				Content: pulumi.String("hello"),
			})
			return err
		})

		return nil
	})
}
```

Correct approach — pass the Output directly:

```go
package main

import (
	"github.com/pulumi/pulumi-aws/sdk/v7/go/aws/s3"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
	pulumi.Run(func(ctx *pulumi.Context) error {
		bucket, err := s3.NewBucket(ctx, "bucket", &s3.BucketArgs{})
		if err != nil {
			return err
		}

		// Pass the output directly - Pulumi handles the dependency
		_, err = s3.NewBucketObject(ctx, "object", &s3.BucketObjectArgs{
			Bucket:  bucket.ID(),  // Output works here
			Content: pulumi.String("hello"),
		})
		if err != nil {
			return err
		}

		return nil
	})
}
```

## Practice 2: Pass Outputs Directly as Inputs

Wrong approach — extracting values breaks the dependency chain:

```go
vpc, err := ec2.NewVpc(ctx, "vpc", &ec2.VpcArgs{
	CidrBlock: pulumi.String("10.0.0.0/16"),
})
if err != nil {
	return err
}

// WRONG: Extracting the value breaks the dependency chain
var vpcId string
vpc.ID().ApplyT(func(id string) string {
	vpcId = id
	return id
})

_, err = ec2.NewSubnet(ctx, "subnet", &ec2.SubnetArgs{
	VpcId:     pulumi.String(vpcId),  // May be empty, no tracked dependency
	CidrBlock: pulumi.String("10.0.1.0/24"),
})
```

Correct approach — pass Output objects directly:

```go
vpc, err := ec2.NewVpc(ctx, "vpc", &ec2.VpcArgs{
	CidrBlock: pulumi.String("10.0.0.0/16"),
})
if err != nil {
	return err
}

_, err = ec2.NewSubnet(ctx, "subnet", &ec2.SubnetArgs{
	VpcId:     vpc.ID(),  // Pass the Output directly
	CidrBlock: pulumi.String("10.0.1.0/24"),
})
if err != nil {
	return err
}
```

String interpolation with Outputs — use `pulumi.Sprintf` instead of `ApplyT` with `fmt.Sprintf`:

```go
// WRONG
name := bucket.ID().ApplyT(func(id string) string {
	return fmt.Sprintf("prefix-%s-suffix", id)
})

// RIGHT - use pulumi.Sprintf for string interpolation with outputs
name := pulumi.Sprintf("prefix-%s-suffix", bucket.ID())
```

## Practice 3: Use Components for Related Resources

Wrong approach — flat structure with no logical grouping:

```go
// Flat structure - no logical grouping, hard to reuse
bucket, err := s3.NewBucket(ctx, "app-bucket", &s3.BucketArgs{})
if err != nil {
	return err
}
_, err = s3.NewBucketPolicy(ctx, "app-bucket-policy", &s3.BucketPolicyArgs{
	Bucket: bucket.ID(),
	Policy: pulumi.String(policyDoc),
})
if err != nil {
	return err
}
_, err = cloudfront.NewOriginAccessIdentity(ctx, "app-oai", &cloudfront.OriginAccessIdentityArgs{})
if err != nil {
	return err
}
_, err = cloudfront.NewDistribution(ctx, "app-cdn", &cloudfront.DistributionArgs{ /* ... */ })
if err != nil {
	return err
}
```

Correct approach — group related resources in a component struct:

```go
type StaticSiteArgs struct {
	Domain  pulumi.StringInput `pulumi:"domain"`
	Content pulumi.AssetArchive `pulumi:"content"`
}

type StaticSite struct {
	pulumi.ResourceState
	Url pulumi.StringOutput `pulumi:"url"`
}

func NewStaticSite(ctx *pulumi.Context, name string, args *StaticSiteArgs, opts ...pulumi.ResourceOption) (*StaticSite, error) {
	component := &StaticSite{}
	err := ctx.RegisterComponentResource("myorg:components:StaticSite", name, component, opts...)
	if err != nil {
		return nil, err
	}

	// Resources created here - see practice 4 for parent setup
	bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{},
		pulumi.Parent(component))
	if err != nil {
		return nil, err
	}
	// ...

	component.Url = distribution.DomainName
	ctx.RegisterResourceOutputs(component, pulumi.Map{
		"url": component.Url,
	})
	return component, nil
}

// Reusable across stacks
site, err := NewStaticSite(ctx, "marketing", &StaticSiteArgs{
	Domain: pulumi.String("marketing.example.com"),
})
if err != nil {
	return err
}
ctx.Export("url", site.Url)
```

## Practice 4: Always Set `pulumi.Parent(component)` in Components

Wrong approach — no parent means children appear at root level:

```go
func NewMyComponent(ctx *pulumi.Context, name string, opts ...pulumi.ResourceOption) (*MyComponent, error) {
	component := &MyComponent{}
	err := ctx.RegisterComponentResource("myorg:components:MyComponent", name, component, opts...)
	if err != nil {
		return nil, err
	}

	// WRONG: No parent set - this bucket appears at root level
	_, err = s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{})
	if err != nil {
		return nil, err
	}

	return component, nil
}
```

Correct approach — pass `pulumi.Parent(component)` to all child resources:

```go
func NewMyComponent(ctx *pulumi.Context, name string, opts ...pulumi.ResourceOption) (*MyComponent, error) {
	component := &MyComponent{}
	err := ctx.RegisterComponentResource("myorg:components:MyComponent", name, component, opts...)
	if err != nil {
		return nil, err
	}

	// RIGHT: Parent establishes hierarchy
	bucket, err := s3.NewBucket(ctx, name+"-bucket", &s3.BucketArgs{},
		pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	_, err = s3.NewBucketPolicy(ctx, name+"-policy", &s3.BucketPolicyArgs{
		Bucket: bucket.ID(),
		Policy: pulumi.String(policyDoc),
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	return component, nil
}
```

## Practice 5: Encrypt Secrets from Day One

Retrieving and using secrets in Go:

```go
conf := config.New(ctx, "")

// This retrieves a secret - the value stays encrypted
dbPassword := conf.RequireSecret("databasePassword")

// Creating outputs from secrets preserves secrecy
connectionString := pulumi.Sprintf("postgres://user:%s@host/db", dbPassword)
// connectionString is also a secret Output

// Explicitly mark values as secret
computed := pulumi.ToSecret(someValue)
```

## Practice 6: Use Aliases When Refactoring

Wrong approach — renamed without alias destroys the resource:

```go
// Before: resource named "my-bucket"
bucket, err := s3.NewBucket(ctx, "my-bucket", &s3.BucketArgs{})

// After: renamed without alias - DESTROYS THE BUCKET
bucket, err := s3.NewBucket(ctx, "application-bucket", &s3.BucketArgs{})
```

Correct approach — alias preserves resource identity:

```go
// After: renamed with alias - preserves the existing bucket
bucket, err := s3.NewBucket(ctx, "application-bucket", &s3.BucketArgs{},
	pulumi.Aliases([]pulumi.Alias{{Name: pulumi.String("my-bucket")}}))
```

Moving a resource into a component:

```go
// Before: top-level resource
bucket, err := s3.NewBucket(ctx, "my-bucket", &s3.BucketArgs{})

// After: inside a component - needs alias with old parent
func NewMyComponent(ctx *pulumi.Context, name string, opts ...pulumi.ResourceOption) (*MyComponent, error) {
	component := &MyComponent{}
	err := ctx.RegisterComponentResource("myorg:components:MyComponent", name, component, opts...)
	if err != nil {
		return nil, err
	}

	bucket, err := s3.NewBucket(ctx, "bucket", &s3.BucketArgs{},
		pulumi.Parent(component),
		pulumi.Aliases([]pulumi.Alias{{
			Name:   pulumi.String("my-bucket"),
			Parent: pulumi.RootStackResource,  // Was at root
		}}))
	if err != nil {
		return nil, err
	}

	return component, nil
}
```

Alias types:

```go
// Simple name change
pulumi.Aliases([]pulumi.Alias{{Name: pulumi.String("old-name")}})

// Parent change
pulumi.Aliases([]pulumi.Alias{{Name: pulumi.String("resource-name"), Parent: oldParent}})

// Full URN (when you know the exact previous URN)
pulumi.Aliases([]pulumi.Alias{{Name: pulumi.String("urn:pulumi:stack::project::aws:s3/bucket:Bucket::old-name")}})
```
