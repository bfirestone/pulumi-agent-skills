# Python Examples: Pulumi Best Practices

## Practice 1: Never Create Resources Inside `apply()`

Wrong approach — resource won't appear in preview:

```python
import pulumi
import pulumi_aws as aws

bucket = aws.s3.Bucket("bucket")

# WRONG: This resource won't appear in preview
bucket.id.apply(lambda bucket_id:
    aws.s3.BucketObject("object",
        bucket=bucket_id,
        content="hello",
    )
)
```

Correct approach — pass the Output directly:

```python
import pulumi
import pulumi_aws as aws

bucket = aws.s3.Bucket("bucket")

# Pass the output directly - Pulumi handles the dependency
obj = aws.s3.BucketObject("object",
    bucket=bucket.id,  # Output[str] works here
    content="hello",
)
```

## Practice 2: Pass Outputs Directly as Inputs

Wrong approach — extracting values breaks the dependency chain:

```python
import pulumi
import pulumi_aws as aws

vpc = aws.ec2.Vpc("vpc", cidr_block="10.0.0.0/16")

# WRONG: Extracting the value breaks the dependency chain
vpc_id_value = None
def capture(id):
    global vpc_id_value
    vpc_id_value = id
vpc.id.apply(capture)

subnet = aws.ec2.Subnet("subnet",
    vpc_id=vpc_id_value,  # May be None, no tracked dependency
    cidr_block="10.0.1.0/24",
)
```

Correct approach — pass Output objects directly:

```python
import pulumi
import pulumi_aws as aws

vpc = aws.ec2.Vpc("vpc", cidr_block="10.0.0.0/16")

subnet = aws.ec2.Subnet("subnet",
    vpc_id=vpc.id,  # Pass the Output directly
    cidr_block="10.0.1.0/24",
)
```

String interpolation with Outputs — in Python, **never use f-strings directly on Output objects**. The f-string silently calls `__str__()` on the Output, producing a garbage representation instead of the resolved value.

Wrong approach:

```python
# WRONG - f-string silently calls __str__() on Output, producing garbage
name = f"prefix-{bucket.id}-suffix"
```

Correct approaches:

```python
# RIGHT - use pulumi.Output.concat for simple cases
name = pulumi.Output.concat("prefix-", bucket.id, "-suffix")

# RIGHT - use .apply() for complex transformations
name = bucket.id.apply(lambda id: f"prefix-{id}-suffix")
```

## Practice 3: Use Components for Related Resources

Wrong approach — flat structure with no logical grouping:

```python
import pulumi_aws as aws

# Flat structure - no logical grouping, hard to reuse
bucket = aws.s3.Bucket("app-bucket")
bucket_policy = aws.s3.BucketPolicy("app-bucket-policy",
    bucket=bucket.id,
    policy=policy_doc,
)
origin_access_identity = aws.cloudfront.OriginAccessIdentity("app-oai")
distribution = aws.cloudfront.Distribution("app-cdn", ...)
```

Correct approach — group related resources in a ComponentResource class:

```python
import pulumi
import pulumi_aws as aws


class StaticSiteArgs:
    def __init__(self, domain: str, content: pulumi.asset.AssetArchive):
        self.domain = domain
        self.content = content


class StaticSite(pulumi.ComponentResource):
    url: pulumi.Output[str]

    def __init__(self, name: str, args: StaticSiteArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:components:StaticSite", name, None, opts)

        # Resources created here - see practice 4 for parent setup
        bucket = aws.s3.Bucket(f"{name}-bucket",
            opts=pulumi.ResourceOptions(parent=self))
        # ...

        self.url = distribution.domain_name
        self.register_outputs({"url": self.url})


# Reusable across stacks
site = StaticSite("marketing", StaticSiteArgs(
    domain="marketing.example.com",
    content=pulumi.asset.FileArchive("./dist"),
))
```

## Practice 4: Always Set `parent=self` in Components

Wrong approach — no parent means children appear at root level:

```python
class MyComponent(pulumi.ComponentResource):
    def __init__(self, name: str, opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:components:MyComponent", name, None, opts)

        # WRONG: No parent set - this bucket appears at root level
        bucket = aws.s3.Bucket(f"{name}-bucket")
```

Correct approach — pass `opts=pulumi.ResourceOptions(parent=self)` to all child resources:

```python
class MyComponent(pulumi.ComponentResource):
    def __init__(self, name: str, opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:components:MyComponent", name, None, opts)

        # RIGHT: Parent establishes hierarchy
        bucket = aws.s3.Bucket(f"{name}-bucket",
            opts=pulumi.ResourceOptions(parent=self))

        policy = aws.s3.BucketPolicy(f"{name}-policy",
            bucket=bucket.id,
            policy=policy_doc,
            opts=pulumi.ResourceOptions(parent=self))
```

## Practice 5: Encrypt Secrets from Day One

Retrieving and using secrets in Python:

```python
import pulumi

config = pulumi.Config()

# This retrieves a secret - the value stays encrypted
db_password = config.require_secret("databasePassword")

# Creating outputs from secrets preserves secrecy
connection_string = db_password.apply(
    lambda pw: f"postgres://user:{pw}@host/db"
)
# connection_string is also a secret Output

# Explicitly mark values as secret
computed = pulumi.Output.secret(some_value)
```

## Practice 6: Use Aliases When Refactoring

Wrong approach — renamed without alias destroys the resource:

```python
# Before: resource named "my-bucket"
bucket = aws.s3.Bucket("my-bucket")

# After: renamed without alias - DESTROYS THE BUCKET
bucket = aws.s3.Bucket("application-bucket")
```

Correct approach — alias preserves resource identity:

```python
# After: renamed with alias - preserves the existing bucket
bucket = aws.s3.Bucket("application-bucket",
    opts=pulumi.ResourceOptions(
        aliases=[pulumi.Alias(name="my-bucket")],
    ))
```

Moving a resource into a component:

```python
# Before: top-level resource
bucket = aws.s3.Bucket("my-bucket")

# After: inside a component - needs alias with old parent
class MyComponent(pulumi.ComponentResource):
    def __init__(self, name: str, opts: pulumi.ResourceOptions = None):
        super().__init__("myorg:components:MyComponent", name, None, opts)

        bucket = aws.s3.Bucket("bucket",
            opts=pulumi.ResourceOptions(
                parent=self,
                aliases=[pulumi.Alias(
                    name="my-bucket",
                    parent=pulumi.ROOT_STACK_RESOURCE,  # Was at root
                )],
            ))
```

Alias types:

```python
# Simple name change
opts=pulumi.ResourceOptions(aliases=[pulumi.Alias(name="old-name")])

# Parent change
opts=pulumi.ResourceOptions(aliases=[
    pulumi.Alias(name="resource-name", parent=old_parent)
])

# Full URN (when you know the exact previous URN)
opts=pulumi.ResourceOptions(aliases=[
    "urn:pulumi:stack::project::aws:s3/bucket:Bucket::old-name"
])
```
