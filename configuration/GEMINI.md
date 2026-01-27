# Pulumi Configuration Skills

This extension provides skills for managing secrets, configuration, and credentials with Pulumi ESC (Environments, Secrets, and Configuration).

## Included Skills

- **esc**: Guidance for working with Pulumi ESC, including managing secrets, configuration, OIDC credentials, and integrating with secret stores like AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, and 1Password

Skills are activated automatically when relevant configuration management tasks are detected.

## MCP Server

This extension includes the Pulumi MCP server, which provides access to your Pulumi Cloud organization data, including ESC environments and configuration.

The MCP server requires the `PULUMI_ACCESS_TOKEN` environment variable. Create an access token at https://app.pulumi.com/account/tokens and set it in your environment.
