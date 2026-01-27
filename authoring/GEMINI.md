# Pulumi Authoring Skills

This extension provides skills for writing quality Pulumi programs, components, automation, and secrets management with ESC.

## Included Skills

- **pulumi-best-practices**: Best practices for writing reliable Pulumi programs
- **pulumi-component**: Guide for authoring ComponentResource classes
- **pulumi-automation-api**: Best practices for using Pulumi Automation API
- **esc**: Guidance for working with Pulumi ESC (Environments, Secrets, and Configuration)

Skills are activated automatically when relevant authoring tasks are detected.

## MCP Server

This extension includes the Pulumi MCP server, which provides access to your Pulumi Cloud organization data, cross-stack visibility through Pulumi Insights, and registry schema information.

The MCP server requires the `PULUMI_ACCESS_TOKEN` environment variable. Create an access token at https://app.pulumi.com/account/tokens and set it in your environment.
