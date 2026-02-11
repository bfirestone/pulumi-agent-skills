# TypeScript Examples: Pulumi Best Practices

## Practice 1: Never Create Resources Inside `apply()`

Wrong approach — resource won't appear in preview:

```typescript
const bucket = new aws.s3.Bucket("bucket");

bucket.id.apply(bucketId => {
    // WRONG: This resource won't appear in preview
    new aws.s3.BucketObject("object", {
        bucket: bucketId,
        content: "hello",
    });
});
```

Correct approach — pass the Output directly:

```typescript
const bucket = new aws.s3.Bucket("bucket");

// Pass the output directly - Pulumi handles the dependency
const object = new aws.s3.BucketObject("object", {
    bucket: bucket.id,  // Output<string> works here
    content: "hello",
});
```

## Practice 2: Pass Outputs Directly as Inputs

Wrong approach — extracting values breaks the dependency chain:

```typescript
const vpc = new aws.ec2.Vpc("vpc", { cidrBlock: "10.0.0.0/16" });

// WRONG: Extracting the value breaks the dependency chain
let vpcId: string;
vpc.id.apply(id => { vpcId = id; });

const subnet = new aws.ec2.Subnet("subnet", {
    vpcId: vpcId,  // May be undefined, no tracked dependency
    cidrBlock: "10.0.1.0/24",
});
```

Correct approach — pass Output objects directly:

```typescript
const vpc = new aws.ec2.Vpc("vpc", { cidrBlock: "10.0.0.0/16" });

const subnet = new aws.ec2.Subnet("subnet", {
    vpcId: vpc.id,  // Pass the Output directly
    cidrBlock: "10.0.1.0/24",
});
```

String interpolation with Outputs:

```typescript
// WRONG
const name = bucket.id.apply(id => `prefix-${id}-suffix`);

// RIGHT - use pulumi.interpolate for template literals
const name = pulumi.interpolate`prefix-${bucket.id}-suffix`;

// RIGHT - use pulumi.concat for simple concatenation
const name = pulumi.concat("prefix-", bucket.id, "-suffix");
```

## Practice 3: Use Components for Related Resources

Wrong approach — flat structure with no logical grouping:

```typescript
// Flat structure - no logical grouping, hard to reuse
const bucket = new aws.s3.Bucket("app-bucket");
const bucketPolicy = new aws.s3.BucketPolicy("app-bucket-policy", {
    bucket: bucket.id,
    policy: policyDoc,
});
const originAccessIdentity = new aws.cloudfront.OriginAccessIdentity("app-oai");
const distribution = new aws.cloudfront.Distribution("app-cdn", { /* ... */ });
```

Correct approach — group related resources in a ComponentResource:

```typescript
interface StaticSiteArgs {
    domain: string;
    content: pulumi.asset.AssetArchive;
}

class StaticSite extends pulumi.ComponentResource {
    public readonly url: pulumi.Output<string>;

    constructor(name: string, args: StaticSiteArgs, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:components:StaticSite", name, args, opts);

        // Resources created here - see practice 4 for parent setup
        const bucket = new aws.s3.Bucket(`${name}-bucket`, {}, { parent: this });
        // ...

        this.url = distribution.domainName;
        this.registerOutputs({ url: this.url });
    }
}

// Reusable across stacks
const site = new StaticSite("marketing", {
    domain: "marketing.example.com",
    content: new pulumi.asset.FileArchive("./dist"),
});
```

## Practice 4: Always Set `parent: this` in Components

Wrong approach — no parent means children appear at root level:

```typescript
class MyComponent extends pulumi.ComponentResource {
    constructor(name: string, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:components:MyComponent", name, {}, opts);

        // WRONG: No parent set - this bucket appears at root level
        const bucket = new aws.s3.Bucket(`${name}-bucket`);
    }
}
```

Correct approach — set `parent: this` on all child resources:

```typescript
class MyComponent extends pulumi.ComponentResource {
    constructor(name: string, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:components:MyComponent", name, {}, opts);

        // RIGHT: Parent establishes hierarchy
        const bucket = new aws.s3.Bucket(`${name}-bucket`, {}, {
            parent: this
        });

        const policy = new aws.s3.BucketPolicy(`${name}-policy`, {
            bucket: bucket.id,
            policy: policyDoc,
        }, {
            parent: this
        });
    }
}
```

## Practice 5: Encrypt Secrets from Day One

Retrieving and using secrets in TypeScript:

```typescript
const config = new pulumi.Config();

// This retrieves a secret - the value stays encrypted
const dbPassword = config.requireSecret("databasePassword");

// Creating outputs from secrets preserves secrecy
const connectionString = pulumi.interpolate`postgres://user:${dbPassword}@host/db`;
// connectionString is also a secret Output

// Explicitly mark values as secret
const computed = pulumi.secret(someValue);
```

## Practice 6: Use Aliases When Refactoring

Wrong approach — renamed without alias destroys the resource:

```typescript
// Before: resource named "my-bucket"
const bucket = new aws.s3.Bucket("my-bucket");

// After: renamed without alias - DESTROYS THE BUCKET
const bucket = new aws.s3.Bucket("application-bucket");
```

Correct approach — alias preserves resource identity:

```typescript
// After: renamed with alias - preserves the existing bucket
const bucket = new aws.s3.Bucket("application-bucket", {}, {
    aliases: [{ name: "my-bucket" }],
});
```

Moving a resource into a component:

```typescript
// Before: top-level resource
const bucket = new aws.s3.Bucket("my-bucket");

// After: inside a component - needs alias with old parent
class MyComponent extends pulumi.ComponentResource {
    constructor(name: string, opts?: pulumi.ComponentResourceOptions) {
        super("myorg:components:MyComponent", name, {}, opts);

        const bucket = new aws.s3.Bucket("bucket", {}, {
            parent: this,
            aliases: [{
                name: "my-bucket",
                parent: pulumi.rootStackResource,  // Was at root
            }],
        });
    }
}
```

Alias types:

```typescript
// Simple name change
aliases: [{ name: "old-name" }]

// Parent change
aliases: [{ name: "resource-name", parent: oldParent }]

// Full URN (when you know the exact previous URN)
aliases: ["urn:pulumi:stack::project::aws:s3/bucket:Bucket::old-name"]
```
