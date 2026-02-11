# Go Examples: CloudFormation to Pulumi

## Resource Naming

```go
// CloudFormation:
// "MyAppBucketABC123": { "Type": "AWS::S3::Bucket", ... }

// Pulumi - CORRECT:
myAppBucket, err := s3.NewBucket(ctx, "MyAppBucketABC123", &s3.BucketArgs{ /* ... */ })
if err != nil {
    return err
}

// Pulumi - WRONG (DO NOT do this - import will fail):
myAppBucket, err := s3.NewBucket(ctx, "my-app-bucket", &s3.BucketArgs{ /* ... */ })
```

## Intrinsic Functions

| CloudFormation | Pulumi Go Equivalent |
|----------------|----------------------|
| `!Ref` (resource) | Resource output (e.g., `bucket.ID()`) |
| `!Ref` (parameter) | Pulumi config |
| `!GetAtt Resource.Attr` | Resource property output |
| `!Sub "..."` | `pulumi.Sprintf(...)` |
| `!Join [delim, [...]]` | `pulumi.Sprintf(...)` or `.ApplyT()` |
| `!If [cond, true, false]` | Go `if` statement |
| `!Equals [a, b]` | `==` comparison |
| `!Select [idx, list]` | Array indexing with `.ApplyT()` |
| `!Split [delim, str]` | `.ApplyT(func(v string) []string { return strings.Split(v, delim) })` |
| `Fn::ImportValue` | Stack references or config |

### !Sub

```go
// CloudFormation: !Sub "arn:aws:s3:::${MyBucket}/*"
// Pulumi:
bucketArn := pulumi.Sprintf("arn:aws:s3:::%s/*", myBucket.Bucket)
```

### !GetAtt

```go
// CloudFormation: !GetAtt MyFunction.Arn
// Pulumi:
functionArn := myFunction.Arn
```

## Conditions

```go
// CloudFormation:
// "Conditions": {
//   "CreateProdResources": { "Fn::Equals": [{ "Ref": "Environment" }, "prod"] }
// }

// Pulumi:
conf := config.New(ctx, "")
environment := conf.Require("environment")
createProdResources := environment == "prod"

if createProdResources {
    // Create production-only resources
}
```

## Parameters

```go
// CloudFormation:
// "Parameters": {
//   "InstanceType": { "Type": "String", "Default": "t3.micro" }
// }

// Pulumi:
conf := config.New(ctx, "")
instanceType := conf.Get("instanceType")
if instanceType == "" {
    instanceType = "t3.micro"
}
```

## Mappings

```go
// CloudFormation:
// "Mappings": {
//   "RegionMap": {
//     "us-east-1": { "AMI": "ami-12345" },
//     "us-west-2": { "AMI": "ami-67890" }
//   }
// }

// Pulumi:
type regionConfig struct {
    AMI string
}

regionMap := map[string]regionConfig{
    "us-east-1": {AMI: "ami-12345"},
    "us-west-2": {AMI: "ami-67890"},
}
ami := regionMap[region].AMI
```

## Output Handling

aws-native outputs may return pointer types. Handle nil safely:

```go
// WRONG
functionName := lambdaFunction.FunctionName

// CORRECT - Handle nil pointer safely
functionName := lambdaFunction.FunctionName.ApplyT(func(name *string) string {
    if name != nil {
        return *name
    }
    return ""
}).(pulumi.StringOutput)
```
