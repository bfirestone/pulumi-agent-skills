# Pulumi Migration Skills

This extension provides skills for converting and importing infrastructure from other tools to Pulumi.

## Included Skills

- **pulumi-terraform-to-pulumi**: Migrate Terraform projects to Pulumi
- **pulumi-cdk-to-pulumi**: Convert AWS CDK applications to Pulumi
- **pulumi-cdk-convert**: Automated CDK conversion using the cdk2pulumi tool
- **pulumi-cdk-importer**: Automated import of CDK-managed AWS infrastructure
- **pulumi-arm-to-pulumi**: Convert Azure ARM templates and Bicep to Pulumi
- **pulumi-arm-import**: Import existing Azure resources into Pulumi
- **pulumi-cloudformation-id-lookup**: Look up CloudFormation import identifiers

Skills are activated automatically when relevant migration tasks are detected.

## MCP Server

This extension includes the Pulumi MCP server, which provides access to your Pulumi Cloud organization data, cross-stack visibility through Pulumi Insights, and live infrastructure context.

The MCP server requires the `PULUMI_ACCESS_TOKEN` environment variable. Create an access token at https://app.pulumi.com/account/tokens and set it in your environment.
