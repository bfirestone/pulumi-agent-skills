# Go Examples: ARM to Pulumi Migration

## Basic Resource Conversion

**ARM `Microsoft.Storage/storageAccounts` → Pulumi Go:**

```go
package main

import (
	"github.com/pulumi/pulumi-azure-native/sdk/v2/go/azure/storage"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi/config"
)

func main() {
	pulumi.Run(func(ctx *pulumi.Context) error {
		conf := config.New(ctx, "")
		storageAccountName := conf.Require("storageAccountName")
		location := conf.Require("location")
		resourceGroupName := conf.Require("resourceGroupName")

		_, err := storage.NewStorageAccount(ctx, "storageAccount", &storage.StorageAccountArgs{
			AccountName:       pulumi.String(storageAccountName),
			Location:          pulumi.String(location),
			ResourceGroupName: pulumi.String(resourceGroupName),
			Sku: &storage.SkuArgs{
				Name: pulumi.String("Standard_LRS"),
			},
			Kind:                   pulumi.String("StorageV2"),
			EnableHttpsTrafficOnly: pulumi.Bool(true),
		})
		if err != nil {
			return err
		}

		return nil
	})
}
```

## ARM Parameters → Pulumi Config

```go
conf := config.New(ctx, "")
location := conf.Get("location")
if location == "" {
	location = "eastus"
}
instanceCount := conf.GetInt("instanceCount")
if instanceCount == 0 {
	instanceCount = 2
}
enableBackup := true
if v, err := conf.TryBool("enableBackup"); err == nil {
	enableBackup = v
}
secretValue := conf.RequireSecret("secretValue") // Returns pulumi.StringOutput
```

## ARM Variables → Pulumi Variables

```go
conf := config.New(ctx, "")
prefix := conf.Require("prefix")
resourceGroupId := conf.Require("resourceGroupId")

// Simple variable
webAppName := prefix + "-webapp"

// For uniqueString equivalent, use stack name or generate hash
storageAccountName := strings.ToLower("storage" + resourceGroupId)
```

## ARM Copy Loops → Pulumi Loops

```go
conf := config.New(ctx, "")
subnetCount := conf.GetInt("subnetCount")
if subnetCount == 0 {
	subnetCount = 3
}

for i := 0; i < subnetCount; i++ {
	_, err := network.NewSubnet(ctx, fmt.Sprintf("subnet-%d", i), &network.SubnetArgs{
		SubnetName:         pulumi.Sprintf("subnet-%d", i),
		VirtualNetworkName: vnet.Name,
		ResourceGroupName:  resourceGroup.Name,
		AddressPrefix:      pulumi.Sprintf("10.0.%d.0/24", i),
	})
	if err != nil {
		return err
	}
}
```

## ARM Conditional Resources → Pulumi Conditionals

```go
conf := config.New(ctx, "")
createPublicIP := false
if v, err := conf.TryBool("createPublicIP"); err == nil {
	createPublicIP = v
}

var publicIP *network.PublicIPAddress
if createPublicIP {
	var err error
	publicIP, err = network.NewPublicIPAddress(ctx, "publicIP", &network.PublicIPAddressArgs{
		PublicIpAddressName: pulumi.String(publicIPName),
		Location:            pulumi.String(location),
		ResourceGroupName:   resourceGroup.Name,
	})
	if err != nil {
		return err
	}
}

// Handle optional references
var publicIPId pulumi.StringInput
if publicIP != nil {
	publicIPId = publicIP.ID()
}
```

## ARM DependsOn → Pulumi Dependencies

```go
// Implicit dependency (preferred)
webApp, err := web.NewWebApp(ctx, "webApp", &web.WebAppArgs{
	Name:              pulumi.String(webAppName),
	ResourceGroupName: resourceGroup.Name,
	ServerFarmId:      appServicePlan.ID(), // Implicit dependency through property reference
})
if err != nil {
	return err
}

// Explicit dependency (when needed)
webApp, err := web.NewWebApp(ctx, "webApp", &web.WebAppArgs{
	Name:              pulumi.String(webAppName),
	ResourceGroupName: resourceGroup.Name,
	ServerFarmId:      appServicePlan.ID(),
}, pulumi.DependsOn([]pulumi.Resource{appServicePlan})) // Explicit dependency
if err != nil {
	return err
}
```

## Nested Templates → Pulumi Component Resources

```go
type NetworkComponent struct {
	pulumi.ResourceState
	VpcId   pulumi.StringOutput      `pulumi:"vpcId"`
	Subnets []*network.Subnet
}

type NetworkComponentArgs struct {
	VnetName          string
	ResourceGroupName pulumi.StringInput
	Location          pulumi.StringInput
	AddressPrefix     string
	SubnetConfigs     []SubnetConfig
}

type SubnetConfig struct {
	Name          string
	AddressPrefix string
}

func NewNetworkComponent(ctx *pulumi.Context, name string, args *NetworkComponentArgs, opts ...pulumi.ResourceOption) (*NetworkComponent, error) {
	component := &NetworkComponent{}
	err := ctx.RegisterComponentResource("custom:azure:NetworkComponent", name, component, opts...)
	if err != nil {
		return nil, err
	}

	vnet, err := network.NewVirtualNetwork(ctx, name+"-vnet", &network.VirtualNetworkArgs{
		VirtualNetworkName: pulumi.String(args.VnetName),
		ResourceGroupName:  args.ResourceGroupName,
		Location:           args.Location,
		AddressSpace: &network.AddressSpaceArgs{
			AddressPrefixes: pulumi.StringArray{pulumi.String(args.AddressPrefix)},
		},
	}, pulumi.Parent(component))
	if err != nil {
		return nil, err
	}

	for i, subnetCfg := range args.SubnetConfigs {
		subnet, err := network.NewSubnet(ctx, fmt.Sprintf("%s-subnet-%d", name, i), &network.SubnetArgs{
			SubnetName:         pulumi.String(subnetCfg.Name),
			VirtualNetworkName: vnet.Name,
			ResourceGroupName:  args.ResourceGroupName,
			AddressPrefix:      pulumi.String(subnetCfg.AddressPrefix),
		}, pulumi.Parent(component))
		if err != nil {
			return nil, err
		}
		component.Subnets = append(component.Subnets, subnet)
	}

	component.VpcId = vnet.ID()
	ctx.RegisterResourceOutputs(component, pulumi.Map{
		"vpcId": vnet.ID(),
	})

	return component, nil
}
```

## ARM Outputs → Pulumi Exports

```go
ctx.Export("storageAccountName", storageAccount.Name)
ctx.Export("storageAccountId", storageAccount.ID())
```

## Azure Classic Provider Examples

### Virtual Network

```go
import (
	"github.com/pulumi/pulumi-azure/sdk/v6/go/azure/network"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

vnet, err := network.NewVirtualNetwork(ctx, "vnet", &network.VirtualNetworkArgs{
	Name:              pulumi.String(vnetName),
	Location:          pulumi.String(location),
	ResourceGroupName: pulumi.String(resourceGroupName),
	AddressSpaces:     pulumi.StringArray{pulumi.String("10.0.0.0/16")},
	Subnets: network.VirtualNetworkSubnetArray{
		&network.VirtualNetworkSubnetArgs{
			Name:          pulumi.String("default"),
			AddressPrefix: pulumi.String("10.0.1.0/24"),
		},
		&network.VirtualNetworkSubnetArgs{
			Name:          pulumi.String("apps"),
			AddressPrefix: pulumi.String("10.0.2.0/24"),
		},
	},
})
if err != nil {
	return err
}
```

### App Service Plan and Web App

```go
import (
	"github.com/pulumi/pulumi-azure/sdk/v6/go/azure/appservice"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

appServicePlan, err := appservice.NewServicePlan(ctx, "appServicePlan", &appservice.ServicePlanArgs{
	Name:              pulumi.String(appServicePlanName),
	Location:          pulumi.String(location),
	ResourceGroupName: pulumi.String(resourceGroupName),
	OsType:            pulumi.String("Linux"),
	SkuName:           pulumi.String("B1"),
})
if err != nil {
	return err
}

_, err = appservice.NewLinuxWebApp(ctx, "webApp", &appservice.LinuxWebAppArgs{
	Name:              pulumi.String(webAppName),
	Location:          pulumi.String(location),
	ResourceGroupName: pulumi.String(resourceGroupName),
	ServicePlanId:     appServicePlan.ID(),
	SiteConfig: &appservice.LinuxWebAppSiteConfigArgs{
		ApplicationStack: &appservice.LinuxWebAppSiteConfigApplicationStackArgs{
			NodeVersion: pulumi.String("18-lts"),
		},
	},
	AppSettings: pulumi.StringMap{
		"WEBSITE_NODE_DEFAULT_VERSION": pulumi.String("~18"),
	},
})
if err != nil {
	return err
}
```

## Go Output Handling

Azure Native outputs may return pointer types. Handle nil safely:

```go
// WRONG - Will cause nil pointer dereference
webAppUrl := webApp.DefaultHostName

// CORRECT - Handle nil pointer safely
webAppUrl := webApp.DefaultHostName.ApplyT(func(hostname *string) string {
	if hostname != nil {
		return fmt.Sprintf("https://%s", *hostname)
	}
	return ""
}).(pulumi.StringOutput)
```

## Resource Naming Conventions

ARM template `name` property maps to specific naming fields in Pulumi:

```go
// ARM: "name": "myStorageAccount"
// Pulumi:
storageAccount, err := storage.NewStorageAccount(ctx, "logicalName", &storage.StorageAccountArgs{
	AccountName: pulumi.String("mystorageaccount"), // Actual Azure resource name
	// ...
})
```
