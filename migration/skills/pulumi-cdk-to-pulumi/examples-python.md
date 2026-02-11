# Python Examples: CDK to Pulumi Migration

## aws-native Output Handling

aws-native outputs often include `None`. Always safely unwrap with `.apply()`:

```python
# WRONG - May be None
function_name = lambda_function.function_name

# CORRECT - Handle None safely
function_name = lambda_function.function_name.apply(lambda name: name or "")
```

## Environment Logic Preservation

Carry forward all conditional behaviors:

```python
if current_env["create_vpc"]:
    # create resources
    pass
else:
    vpc_id = pulumi.Output.from_input(current_env["vpc_id"])
```

## Custom Resource Handler Properties

When replacing CDK custom resources with native Pulumi resources, use these Python property names:

| Handler | Replacement |
|---------|-------------|
| `aws-ecr/auto-delete-images-handler` | `aws.ecr.Repository` with `force_delete=True` |
| `aws-s3/auto-delete-objects-handler` | `aws.s3.Bucket` with `force_destroy=True` |
| `aws-logs/log-retention-handler` | `aws.cloudwatch.LogGroup` with `retention_in_days` |
| `aws-route53/delete-existing-record-set-handler` | `aws.route53.Record` with `allow_overwrite=True` |

## Cross-Account/Region Handlers

- `aws-cloudfront/edge-function` → Use `aws.lambda_.Function` with `region="us-east-1"` provider

## Assets (Static Files)

Use `pulumi.FileArchive` or `pulumi.FileAsset` for static file assets.

Migrated assets are reported as: `static files → pulumi.FileArchive`
