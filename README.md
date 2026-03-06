# Azure Troubleshooting VM Terraform Module

This module deploys a Linux or Windows virtual machine for troubleshooting purposes in Azure. It's designed to be simple to use while allowing extensive customization when needed.

## Features
- Rapid deployment of either Linux or Windows VMs with minimal configuration
- Automatic tagging with deployment date and purpose
- Extracts location & network information from subnet ID
- Customizable VM properties for specific testing scenarios
- Support for static IP address assignment
- Linux VM cloud-init with common network troubleshooting tools
- Option to deploy to an existing resource group
- Optional public IP assignment
- Optional NSG creation on the VM NIC with SSH access from your current public IP

## Usage

### Steps
1. Configure the `azurerm` provider in your root module
2. Login via Azure CLI to the correct tenant
3. Run terraform init
4. Run terraform apply

### Minimal Configuration
In its most simple form, a troubleshooting VM can be deployed with just the subnet ID and OS type:

```terraform
provider "azurerm" {
  features {}
  subscription_id = "00000000-0000-0000-0000-000000000000"
}

module "tshoot_vm" {
  source    = "github.com/Greg-Court/azure-tf-tshoot-vm"
  subnet_id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-vnet-rg-name/providers/Microsoft.Network/virtualNetworks/my-vnet-name/subnets/my-subnet-name"
  os_type   = "linux"  # windows or linux
}
```

### Complete Configuration Example
Here's an example showing all available customization options:

```terraform
provider "azurerm" {
  features {}
  subscription_id = "00000000-0000-0000-0000-000000000000"
}

module "tshoot_vm" {
  source    = "github.com/Greg-Court/azure-tf-tshoot-vm"

  # Required parameters
  subnet_id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-vnet-rg-name/providers/Microsoft.Network/virtualNetworks/my-vnet-name/subnets/my-subnet-name"
  os_type   = "windows"  # windows or linux

  # Optional parameters
  use_existing_rg = true             # Use an existing resource group
  rg_name         = "my-existing-rg" # Resource group name (existing or new)

  vm_name_prefix = "vm-custom"
  vm_size        = "Standard_D2s_v3"
  admin_username = "customadmin"
  admin_password = "CustomP@ssw0rd!"
  zone           = 1

  # OS disk configuration
  os_disk_caching              = "ReadOnly"
  os_disk_storage_account_type = "Premium_LRS"

  # VM image configuration
  source_image_publisher = "MicrosoftWindowsDesktop"
  source_image_offer     = "Windows-10"
  source_image_sku       = "win10-21h2-pro"
  source_image_version   = "latest"

  # Network configuration
  private_ip_address_allocation = "Static"
  private_ip_address            = "10.0.1.10"  # Required when allocation is Static
  enable_public_ip              = true          # Assign a public IP to the VM
  create_nsg                    = true          # Create NSG on NIC allowing SSH from your current IP

  # Security and patching
  patch_mode                        = "Manual"
  bypass_platform_safety_checks     = false
  secure_boot_enabled               = true

  # Linux configuration
  enable_cloud_init = true
  custom_cloud_init = <<-EOT
    #cloud-config
    package_update: true
    packages:
      - nginx
      - postgresql-client
      - tcpdump
  EOT

  # Tags
  tags = {
    Environment = "Testing"
    Backup      = "Daily"
    Owner       = "IT Support"
    PatchGroup  = "Weekly"
  }

  rg_tags = {
    Department = "IT"
    CostCenter = "12345"
  }
}
```

## Required Inputs

| Name | Description | Type | Required |
|------|-------------|------|:--------:|
| subnet_id | The ID of the subnet where the VM will be deployed | `string` | yes |
| os_type | The OS type for the VM. Can be 'linux' or 'windows' | `string` | yes |

## Optional Inputs

| Name | Description | Type | Default |
|------|-------------|------|---------|
| use_existing_rg | Whether to use an existing resource group. If true, rg_name must specify an existing RG. | `bool` | `false` |
| rg_name | Resource group name to use (existing when use_existing_rg is true, or new when false). If not provided for a new RG, a name will be generated. | `string` | `null` |
| vm_name | Custom name for the VM. If provided, this will override the automatically generated name. | `string` | `null` |
| vm_name_prefix | Prefix for the VM name | `string` | `"vm-tshoot"` |
| vm_size | The size of the VM | `string` | `"Standard_B2s"` (Linux) or `"Standard_B2ms"` (Windows) |
| admin_username | The administrator username for the VM | `string` | `"azureadmin"` |
| admin_password | The administrator password for the VM | `string` | `"Password123!"` |
| zone | The availability zone number for the VM | `number` | `null` |
| os_disk_caching | The type of caching to use on the OS disk | `string` | `"ReadWrite"` |
| os_disk_storage_account_type | The storage account type for the OS disk | `string` | `"StandardSSD_LRS"` |
| source_image_publisher | The publisher of the VM image | `string` | `"canonical"` (Linux) or `"MicrosoftWindowsServer"` (Windows) |
| source_image_offer | The offer of the VM image | `string` | `"ubuntu-24_04-lts"` (Linux) or `"WindowsServer"` (Windows) |
| source_image_sku | The SKU of the VM image | `string` | `"server"` (Linux) or `"2022-datacenter-g2"` (Windows) |
| source_image_version | The version of the VM image | `string` | `"latest"` |
| private_ip_address_allocation | The private IP address allocation method | `string` | `"Dynamic"` |
| private_ip_address | The static private IP address to assign when private_ip_address_allocation is 'Static' | `string` | `null` |
| enable_public_ip | Whether to create and assign a public IP address to the VM | `bool` | `false` |
| create_nsg | Whether to create an NSG on the VM NIC allowing SSH (port 22) from your current public IP (fetched via ipify) | `bool` | `false` |
| patch_mode | The patching mode for the VM | `string` | `"AutomaticByPlatform"` |
| bypass_platform_safety_checks | Enable bypass platform safety checks on user schedule | `bool` | `true` |
| secure_boot_enabled | Enable secure boot | `bool` | `false` |
| enable_cloud_init | Enable installation of packages on first boot (Linux only) | `bool` | `true` |
| custom_cloud_init | Custom cloud-init config for Linux VMs (replaces default config) | `string` | `null` |
| tags | A map of tags to assign to the virtual machine & associated resources | `map(string)` | `{}` |
| rg_tags | A map of tags to assign to the resource group | `map(string)` | `{}` |

## Outputs

| Name | Description |
|------|-------------|
| rg_name | The name of the resource group |
| vm_id | The ID of the created VM |
| vm_name | The name of the created VM |
| private_ip_address | The private IP address of the VM |
| public_ip_address | The public IP address of the VM (if enabled) |
| admin_username | The administrator username for the VM |
| admin_password | The administrator password for the VM (sensitive) |
