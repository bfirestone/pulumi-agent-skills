# Python Examples: ARM to Pulumi Migration

## Basic Resource Conversion

**ARM `Microsoft.Storage/storageAccounts` → Pulumi Python:**

```python
import pulumi
import pulumi_azure_native as azure_native

config = pulumi.Config()
storage_account_name = config.require("storageAccountName")
location = config.require("location")
resource_group_name = config.require("resourceGroupName")

storage_account = azure_native.storage.StorageAccount("storageAccount",
    account_name=storage_account_name,
    location=location,
    resource_group_name=resource_group_name,
    sku=azure_native.storage.SkuArgs(
        name=azure_native.storage.SkuName.STANDARD_LRS,
    ),
    kind=azure_native.storage.Kind.STORAGE_V2,
    enable_https_traffic_only=True,
)
```

## ARM Parameters → Pulumi Config

```python
config = pulumi.Config()
location = config.get("location") or "eastus"
instance_count = config.get_int("instanceCount") or 2
enable_backup = config.get_bool("enableBackup")
if enable_backup is None:
    enable_backup = True
secret_value = config.require_secret("secretValue")  # Returns Output[str]
```

## ARM Variables → Pulumi Variables

```python
import pulumi

config = pulumi.Config()
prefix = config.require("prefix")
resource_group_id = config.require("resourceGroupId")

# Simple variable
web_app_name = f"{prefix}-webapp"

# For uniqueString equivalent, use stack name or generate hash
storage_account_name = f"storage{resource_group_id}".lower()
```

## ARM Copy Loops → Pulumi Loops

```python
config = pulumi.Config()
subnet_count = config.get_int("subnetCount") or 3

subnets = []
for i in range(subnet_count):
    subnets.append(azure_native.network.Subnet(f"subnet-{i}",
        subnet_name=f"subnet-{i}",
        virtual_network_name=vnet.name,
        resource_group_name=resource_group.name,
        address_prefix=f"10.0.{i}.0/24",
    ))
```

## ARM Conditional Resources → Pulumi Conditionals

```python
config = pulumi.Config()
create_public_ip = config.get_bool("createPublicIP") or False

public_ip = None
if create_public_ip:
    public_ip = azure_native.network.PublicIPAddress("publicIP",
        public_ip_address_name=public_ip_name,
        location=location,
        resource_group_name=resource_group.name,
    )

# Handle optional references
public_ip_id = public_ip.id if public_ip else None
```

## ARM DependsOn → Pulumi Dependencies

```python
# Implicit dependency (preferred)
web_app = azure_native.web.WebApp("webApp",
    name=web_app_name,
    resource_group_name=resource_group.name,
    server_farm_id=app_service_plan.id,  # Implicit dependency through property reference
)

# Explicit dependency (when needed)
web_app = azure_native.web.WebApp("webApp",
    name=web_app_name,
    resource_group_name=resource_group.name,
    server_farm_id=app_service_plan.id,
    opts=pulumi.ResourceOptions(depends_on=[app_service_plan]),
)
```

## Nested Templates → Pulumi Component Resources

```python
class NetworkComponentArgs:
    def __init__(self,
                 vnet_name: pulumi.Input[str],
                 resource_group_name: pulumi.Input[str],
                 location: pulumi.Input[str],
                 address_prefix: pulumi.Input[str],
                 subnets: list[dict]):
        self.vnet_name = vnet_name
        self.resource_group_name = resource_group_name
        self.location = location
        self.address_prefix = address_prefix
        self.subnets = subnets


class NetworkComponent(pulumi.ComponentResource):
    vnet: azure_native.network.VirtualNetwork
    subnets: list[azure_native.network.Subnet]

    def __init__(self, name: str, args: NetworkComponentArgs,
                 opts: pulumi.ResourceOptions = None):
        super().__init__("custom:azure:NetworkComponent", name, None, opts)

        self.vnet = azure_native.network.VirtualNetwork(f"{name}-vnet",
            virtual_network_name=args.vnet_name,
            resource_group_name=args.resource_group_name,
            location=args.location,
            address_space=azure_native.network.AddressSpaceArgs(
                address_prefixes=[args.address_prefix],
            ),
            opts=pulumi.ResourceOptions(parent=self))

        self.subnets = []
        for i, subnet in enumerate(args.subnets):
            self.subnets.append(azure_native.network.Subnet(f"{name}-subnet-{i}",
                subnet_name=subnet["name"],
                virtual_network_name=self.vnet.name,
                resource_group_name=args.resource_group_name,
                address_prefix=subnet["address_prefix"],
                opts=pulumi.ResourceOptions(parent=self)))

        self.register_outputs({
            "vnet_id": self.vnet.id,
            "subnet_ids": [s.id for s in self.subnets],
        })
```

## ARM Outputs → Pulumi Exports

```python
pulumi.export("storage_account_name", storage_account.name)
pulumi.export("storage_account_id", storage_account.id)
```

## Azure Classic Provider Examples

### Virtual Network

```python
import pulumi
import pulumi_azure as azure

config = pulumi.Config()
vnet_name = config.require("vnetName")
location = config.require("location")
resource_group_name = config.require("resourceGroupName")

vnet = azure.network.VirtualNetwork("vnet",
    name=vnet_name,
    location=location,
    resource_group_name=resource_group_name,
    address_spaces=["10.0.0.0/16"],
    subnets=[
        azure.network.VirtualNetworkSubnetArgs(
            name="default",
            address_prefix="10.0.1.0/24",
        ),
        azure.network.VirtualNetworkSubnetArgs(
            name="apps",
            address_prefix="10.0.2.0/24",
        ),
    ],
)
```

### App Service Plan and Web App

```python
import pulumi
import pulumi_azure as azure

config = pulumi.Config()
app_service_plan_name = config.require("appServicePlanName")
web_app_name = config.require("webAppName")
location = config.require("location")
resource_group_name = config.require("resourceGroupName")

app_service_plan = azure.appservice.ServicePlan("appServicePlan",
    name=app_service_plan_name,
    location=location,
    resource_group_name=resource_group_name,
    os_type="Linux",
    sku_name="B1",
)

web_app = azure.appservice.LinuxWebApp("webApp",
    name=web_app_name,
    location=location,
    resource_group_name=resource_group_name,
    service_plan_id=app_service_plan.id,
    site_config=azure.appservice.LinuxWebAppSiteConfigArgs(
        application_stack=azure.appservice.LinuxWebAppSiteConfigApplicationStackArgs(
            node_version="18-lts",
        ),
    ),
    app_settings={
        "WEBSITE_NODE_DEFAULT_VERSION": "~18",
    },
)
```

## Python Output Handling

Azure Native outputs often include `None`. Always safely unwrap with `.apply()`:

```python
# WRONG - May raise an error
web_app_url = f"https://{web_app.default_host_name}"

# CORRECT - Handle None safely
web_app_url = web_app.default_host_name.apply(
    lambda hostname: f"https://{hostname}" if hostname else ""
)
```

## Resource Naming Conventions

ARM template `name` property maps to specific naming fields in Pulumi:

```python
# ARM: "name": "myStorageAccount"
# Pulumi:
storage_account = azure_native.storage.StorageAccount("logicalName",
    account_name="mystorageaccount",  # Actual Azure resource name
    # ...
)
```
