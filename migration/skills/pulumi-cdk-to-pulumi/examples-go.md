# Go Examples: CDK to Pulumi Migration

## aws-native Output Handling

aws-native outputs may return pointer types. Handle nil safely:

```go
// WRONG - Will cause nil pointer dereference
functionName := lambdaFunction.FunctionName

// CORRECT - Handle nil pointer safely
functionName := lambdaFunction.FunctionName.ApplyT(func(name *string) string {
    if name != nil {
        return *name
    }
    return ""
}).(pulumi.StringOutput)
```

## Environment Logic Preservation

Carry forward all conditional behaviors:

```go
if currentEnv.CreateVpc {
    // create resources
} else {
    vpcId := pulumi.String(currentEnv.VpcId)
    _ = vpcId
}
```

## Custom Resource Handler Properties

When replacing CDK custom resources with native Pulumi resources, use these Go property names:

| Handler | Replacement |
|---------|-------------|
| `aws-ecr/auto-delete-images-handler` | `aws.ecr.Repository` with `ForceDelete: pulumi.Bool(true)` |
| `aws-s3/auto-delete-objects-handler` | `aws.s3.Bucket` with `ForceDestroy: pulumi.Bool(true)` |
| `aws-logs/log-retention-handler` | `aws.cloudwatch.LogGroup` with explicit `RetentionInDays` |
| `aws-route53/delete-existing-record-set-handler` | `aws.route53.Record` with `AllowOverwrite: pulumi.Bool(true)` |

## Cross-Account/Region Handlers

- `aws-cloudfront/edge-function` → Use `aws.lambda.Function` with `region: "us-east-1"`

## Assets (Static Files)

Use `pulumi.NewFileArchive` or `pulumi.NewFileAsset` for static file assets.

Migrated assets are reported as: `static files → pulumi.NewFileArchive`
