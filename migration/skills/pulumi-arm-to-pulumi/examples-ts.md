# TypeScript Examples: ARM to Pulumi Migration

## Basic Resource Conversion

**ARM `Microsoft.Storage/storageAccounts` → Pulumi TypeScript:**

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as azure_native from "@pulumi/azure-native";

const config = new pulumi.Config();
const storageAccountName = config.require("storageAccountName");
const location = config.require("location");
const resourceGroupName = config.require("resourceGroupName");

const storageAccount = new azure_native.storage.StorageAccount("storageAccount", {
    accountName: storageAccountName,
    location: location,
    resourceGroupName: resourceGroupName,
    sku: {
        name: azure_native.storage.SkuName.Standard_LRS,
    },
    kind: azure_native.storage.Kind.StorageV2,
    enableHttpsTrafficOnly: true,
});
```

## ARM Parameters → Pulumi Config

```typescript
const config = new pulumi.Config();
const location = config.get("location") || "eastus";
const instanceCount = config.getNumber("instanceCount") || 2;
const enableBackup = config.getBoolean("enableBackup") ?? true;
const secretValue = config.requireSecret("secretValue"); // Returns Output<string>
```

## ARM Variables → Pulumi Variables

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const prefix = config.require("prefix");
const resourceGroupId = config.require("resourceGroupId");

// Simple variable
const webAppName = `${prefix}-webapp`;

// For uniqueString equivalent, use stack name or generate hash
const storageAccountName = `storage${resourceGroupId}`.toLowerCase();
```

## ARM Copy Loops → Pulumi Loops

```typescript
const config = new pulumi.Config();
const subnetCount = config.getNumber("subnetCount") || 3;

const subnets: azure_native.network.Subnet[] = [];
for (let i = 0; i < subnetCount; i++) {
    subnets.push(new azure_native.network.Subnet(`subnet-${i}`, {
        subnetName: `subnet-${i}`,
        virtualNetworkName: vnet.name,
        resourceGroupName: resourceGroup.name,
        addressPrefix: `10.0.${i}.0/24`,
    }));
}
```

## ARM Conditional Resources → Pulumi Conditionals

```typescript
const config = new pulumi.Config();
const createPublicIP = config.getBoolean("createPublicIP") ?? false;

let publicIP: azure_native.network.PublicIPAddress | undefined;
if (createPublicIP) {
    publicIP = new azure_native.network.PublicIPAddress("publicIP", {
        publicIpAddressName: publicIPName,
        location: location,
        resourceGroupName: resourceGroup.name,
    });
}

// Handle optional references
const publicIPId = publicIP ? publicIP.id : pulumi.output(undefined);
```

## ARM DependsOn → Pulumi Dependencies

```typescript
// Implicit dependency (preferred)
const webApp = new azure_native.web.WebApp("webApp", {
    name: webAppName,
    resourceGroupName: resourceGroup.name,
    serverFarmId: appServicePlan.id, // Implicit dependency through property reference
});

// Explicit dependency (when needed)
const webApp = new azure_native.web.WebApp("webApp", {
    name: webAppName,
    resourceGroupName: resourceGroup.name,
    serverFarmId: appServicePlan.id,
}, {
    dependsOn: [appServicePlan], // Explicit dependency
});
```

## Nested Templates → Pulumi Component Resources

```typescript
class NetworkComponent extends pulumi.ComponentResource {
    public readonly vnet: azure_native.network.VirtualNetwork;
    public readonly subnets: azure_native.network.Subnet[];

    constructor(name: string, args: NetworkComponentArgs, opts?: pulumi.ComponentResourceOptions) {
        super("custom:azure:NetworkComponent", name, {}, opts);

        const defaultOptions = { parent: this };

        this.vnet = new azure_native.network.VirtualNetwork(`${name}-vnet`, {
            virtualNetworkName: args.vnetName,
            resourceGroupName: args.resourceGroupName,
            location: args.location,
            addressSpace: {
                addressPrefixes: [args.addressPrefix],
            },
        }, defaultOptions);

        this.subnets = args.subnets.map((subnet, i) =>
            new azure_native.network.Subnet(`${name}-subnet-${i}`, {
                subnetName: subnet.name,
                virtualNetworkName: this.vnet.name,
                resourceGroupName: args.resourceGroupName,
                addressPrefix: subnet.addressPrefix,
            }, defaultOptions)
        );

        this.registerOutputs({
            vnetId: this.vnet.id,
            subnetIds: this.subnets.map(s => s.id),
        });
    }
}
```

## ARM Outputs → Pulumi Exports

```typescript
export const storageAccountName = storageAccount.name;
export const storageAccountId = storageAccount.id;
```

## Azure Classic Provider Examples

### Virtual Network

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as azure from "@pulumi/azure";

const config = new pulumi.Config();
const vnetName = config.require("vnetName");
const location = config.require("location");
const resourceGroupName = config.require("resourceGroupName");

const vnet = new azure.network.VirtualNetwork("vnet", {
    name: vnetName,
    location: location,
    resourceGroupName: resourceGroupName,
    addressSpaces: ["10.0.0.0/16"],
    subnets: [
        {
            name: "default",
            addressPrefix: "10.0.1.0/24",
        },
        {
            name: "apps",
            addressPrefix: "10.0.2.0/24",
        },
    ],
});
```

### App Service Plan and Web App

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as azure from "@pulumi/azure";

const config = new pulumi.Config();
const appServicePlanName = config.require("appServicePlanName");
const webAppName = config.require("webAppName");
const location = config.require("location");
const resourceGroupName = config.require("resourceGroupName");

const appServicePlan = new azure.appservice.ServicePlan("appServicePlan", {
    name: appServicePlanName,
    location: location,
    resourceGroupName: resourceGroupName,
    osType: "Linux",
    skuName: "B1",
});

const webApp = new azure.appservice.LinuxWebApp("webApp", {
    name: webAppName,
    location: location,
    resourceGroupName: resourceGroupName,
    servicePlanId: appServicePlan.id,
    siteConfig: {
        applicationStack: {
            nodeVersion: "18-lts",
        },
    },
    appSettings: {
        "WEBSITE_NODE_DEFAULT_VERSION": "~18",
    },
});
```

## TypeScript Output Handling

Azure Native outputs often include undefined. Avoid `!` non-null assertions. Always safely unwrap with `.apply()`:

```typescript
// ❌ WRONG - Will cause TypeScript errors
const webAppUrl = `https://${webApp.defaultHostName!}`;

// ✅ CORRECT - Handle undefined safely
const webAppUrl = webApp.defaultHostName.apply(hostname =>
    hostname ? `https://${hostname}` : ""
);
```

## Resource Naming Conventions

ARM template `name` property maps to specific naming fields in Pulumi:

```typescript
// ARM: "name": "myStorageAccount"
// Pulumi:
new azure_native.storage.StorageAccount("logicalName", {
    accountName: "mystorageaccount", // Actual Azure resource name
    // ...
});
```
