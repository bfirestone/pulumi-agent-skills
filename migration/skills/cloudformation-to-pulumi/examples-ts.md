# TypeScript Examples: CloudFormation to Pulumi

## Resource Naming

```typescript
// CloudFormation:
// "MyAppBucketABC123": { "Type": "AWS::S3::Bucket", ... }

// Pulumi - CORRECT:
const myAppBucket = new aws.s3.Bucket("MyAppBucketABC123", { ... });

// Pulumi - WRONG (DO NOT do this - import will fail):
const myAppBucket = new aws.s3.Bucket("my-app-bucket", { ... });
```

## Intrinsic Functions

| CloudFormation | Pulumi TypeScript Equivalent |
|----------------|------------------------------|
| `!Ref` (resource) | Resource output (e.g., `bucket.id`) |
| `!Ref` (parameter) | Pulumi config |
| `!GetAtt Resource.Attr` | Resource property output |
| `!Sub "..."` | `pulumi.interpolate` |
| `!Join [delim, [...]]` | `pulumi.interpolate` or `.apply()` |
| `!If [cond, true, false]` | Ternary operator |
| `!Equals [a, b]` | `===` comparison |
| `!Select [idx, list]` | Array indexing with `.apply()` |
| `!Split [delim, str]` | `.apply(v => v.split(...))` |
| `Fn::ImportValue` | Stack references or config |

### !Sub

```typescript
// CloudFormation: !Sub "arn:aws:s3:::${MyBucket}/*"
// Pulumi:
const bucketArn = pulumi.interpolate`arn:aws:s3:::${myBucket.bucket}/*`;
```

### !GetAtt

```typescript
// CloudFormation: !GetAtt MyFunction.Arn
// Pulumi:
const functionArn = myFunction.arn;
```

## Conditions

```typescript
// CloudFormation:
// "Conditions": {
//   "CreateProdResources": { "Fn::Equals": [{ "Ref": "Environment" }, "prod"] }
// }

// Pulumi:
const config = new pulumi.Config();
const environment = config.require("environment");
const createProdResources = environment === "prod";

if (createProdResources) {
  // Create production-only resources
}
```

## Parameters

```typescript
// CloudFormation:
// "Parameters": {
//   "InstanceType": { "Type": "String", "Default": "t3.micro" }
// }

// Pulumi:
const config = new pulumi.Config();
const instanceType = config.get("instanceType") || "t3.micro";
```

## Mappings

```typescript
// CloudFormation:
// "Mappings": {
//   "RegionMap": {
//     "us-east-1": { "AMI": "ami-12345" },
//     "us-west-2": { "AMI": "ami-67890" }
//   }
// }

// Pulumi:
const regionMap: Record<string, { ami: string }> = {
  "us-east-1": { ami: "ami-12345" },
  "us-west-2": { ami: "ami-67890" },
};
const ami = regionMap[aws.config.region!].ami;
```

## Output Handling

aws-native outputs often include undefined. Avoid `!` non-null assertions. Always safely unwrap with `.apply()`:

```typescript
// WRONG
functionName: lambdaFunction.functionName!,

// CORRECT
functionName: lambdaFunction.functionName.apply(name => name || ""),
```
