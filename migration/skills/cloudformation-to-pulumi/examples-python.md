# Python Examples: CloudFormation to Pulumi

## Resource Naming

```python
# CloudFormation:
# "MyAppBucketABC123": { "Type": "AWS::S3::Bucket", ... }

# Pulumi - CORRECT:
my_app_bucket = aws.s3.Bucket("MyAppBucketABC123", ...)

# Pulumi - WRONG (DO NOT do this - import will fail):
my_app_bucket = aws.s3.Bucket("my-app-bucket", ...)
```

## Intrinsic Functions

| CloudFormation | Pulumi Python Equivalent |
|----------------|--------------------------|
| `!Ref` (resource) | Resource output (e.g., `bucket.id`) |
| `!Ref` (parameter) | Pulumi config |
| `!GetAtt Resource.Attr` | Resource property output |
| `!Sub "..."` | `.apply(lambda v: f"...")` or `Output.concat(...)` |
| `!Join [delim, [...]]` | `Output.concat(...)` or `.apply()` |
| `!If [cond, true, false]` | Ternary expression or `if`/`else` |
| `!Equals [a, b]` | `==` comparison |
| `!Select [idx, list]` | `.apply(lambda v: v[idx])` |
| `!Split [delim, str]` | `.apply(lambda v: v.split(...))` |
| `Fn::ImportValue` | Stack references or config |

### !Sub

```python
# CloudFormation: !Sub "arn:aws:s3:::${MyBucket}/*"
# Pulumi:
bucket_arn = my_bucket.bucket.apply(lambda b: f"arn:aws:s3:::{b}/*")
```

### !GetAtt

```python
# CloudFormation: !GetAtt MyFunction.Arn
# Pulumi:
function_arn = my_function.arn
```

## Conditions

```python
# CloudFormation:
# "Conditions": {
#   "CreateProdResources": { "Fn::Equals": [{ "Ref": "Environment" }, "prod"] }
# }

# Pulumi:
config = pulumi.Config()
environment = config.require("environment")
create_prod_resources = environment == "prod"

if create_prod_resources:
    # Create production-only resources
    pass
```

## Parameters

```python
# CloudFormation:
# "Parameters": {
#   "InstanceType": { "Type": "String", "Default": "t3.micro" }
# }

# Pulumi:
config = pulumi.Config()
instance_type = config.get("instanceType") or "t3.micro"
```

## Mappings

```python
# CloudFormation:
# "Mappings": {
#   "RegionMap": {
#     "us-east-1": { "AMI": "ami-12345" },
#     "us-west-2": { "AMI": "ami-67890" }
#   }
# }

# Pulumi:
region_map: dict[str, dict[str, str]] = {
    "us-east-1": {"ami": "ami-12345"},
    "us-west-2": {"ami": "ami-67890"},
}
ami = region_map[aws.config.region]["ami"]
```

## Output Handling

aws-native outputs often include `None`. Always safely unwrap with `.apply()`:

```python
# WRONG
function_name = lambda_function.function_name  # May be None

# CORRECT
function_name = lambda_function.function_name.apply(lambda name: name or "")
```
