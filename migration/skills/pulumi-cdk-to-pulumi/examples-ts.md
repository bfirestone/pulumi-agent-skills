# TypeScript Examples: CDK to Pulumi Migration

## aws-native Output Handling

aws-native outputs often include undefined. Avoid `!` non-null assertions. Always safely unwrap with `.apply()`:

```ts
// ❌ WRONG - Will cause TypeScript errors
functionName: lambdaFunction.functionName!,

// ✅ CORRECT - Handle undefined safely
functionName: lambdaFunction.functionName.apply(name => name || ""),
```

## Environment Logic Preservation

Carry forward all conditional behaviors:

```ts
if (currentEnv.createVpc) {
  // create resources
} else {
  const vpcId = pulumi.output(currentEnv.vpcId);
}
```

## Custom Resource Handler Properties

When replacing CDK custom resources with native Pulumi resources, use these TypeScript property names:

| Handler | Replacement |
|---------|-------------|
| `aws-ecr/auto-delete-images-handler` | `aws.ecr.Repository` with `forceDelete: true` |
| `aws-s3/auto-delete-objects-handler` | `aws.s3.Bucket` with `forceDestroy: true` |
| `aws-logs/log-retention-handler` | `aws.cloudwatch.LogGroup` with `retentionInDays` |
| `aws-route53/delete-existing-record-set-handler` | `aws.route53.Record` with `allowOverwrite: true` |

## Cross-Account/Region Handlers

- `aws-cloudfront/edge-function` → Use `aws.lambda.Function` with `region: "us-east-1"`

## Assets (Static Files)

Use `pulumi.FileArchive` or `pulumi.FileAsset` for static file assets.

Migrated assets are reported as: `static files → pulumi.FileArchive`
