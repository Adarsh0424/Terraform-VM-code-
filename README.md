**********************************************************************************************
provider.tf
************************************************************************************************
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "4.74.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id ="
}
************************************************************************************************************
Terraform.tfvars
***********************************************************************************************************
resource_groups = {
  rg1 = {
    name     = "tinku-rg"
    location = "centralindia"
  }
  rg2 = {
    name     = "sinku-rg"
    location = "centralus"
  }
}

virtual_networks = {
  vnet1 = {
    name                = "tinku_vnet"
    resource_group_name = "rg1"
    address_space       = ["10.0.0.0/16"]

  }
  vnet2 = {
    name                = "sinku_vnet"
    resource_group_name = "rg2"
    address_space       = ["20.0.0.0/16"]

  }
}

subnets = {

  subnet1 = {
    name                 = "tinku-subnet"
    resource_group_name  = "rg1"
    virtual_network_name = "vnet1"

    address_prefixes = ["10.0.1.0/24"]
  }

  subnet2 = {
    name                 = "sinku-subnet"
    resource_group_name  = "rg2"
    virtual_network_name = "vnet2"

    address_prefixes = ["20.0.1.0/24"]
  }
}

virtual_machines = {
  vm1 = {
    name                = "tinku-vm"
    resource_group_name = "rg1"
    subnet_name         = "subnet1"
    nic_name            = "tinku-nic"
    size                = "Standard_B1ms"
    admin_username      = "adminuser1"
    admin_password      = "admin@123456"
  }
  vm2 = {
    name                = "sinku-vm"
    resource_group_name = "rg2"
    subnet_name         = "subnet2"
    nic_name            = "sinku-nic"
    size                = "Standard_B1ms"
    admin_username      = "adminuser2"
    admin_password      = "admin@123456"
  }
}

*********************************************************************************************************
variable.tf
*******************************************************************************************************
variable "resource_groups" {
}

variable "virtual_networks" {
}
variable "subnets" {

}
variable "virtual_machines" {

}

****************************************************************************************************
main.tf
*******************************************************************************************
resource "azurerm_resource_group" "resource_groups" {
  for_each = var.resource_groups
  name     = each.value.name
  location = each.value.location


}

resource "azurerm_virtual_network" "virtual_networks" {

  for_each = var.virtual_networks

  name = each.value.name

  location = azurerm_resource_group.resource_groups[each.value.resource_group_name].location

  resource_group_name = azurerm_resource_group.resource_groups[each.value.resource_group_name].name

  address_space = each.value.address_space
}
resource "azurerm_subnet" "subnets" {
  for_each            = var.subnets
  name                = each.value.name
  resource_group_name = azurerm_resource_group.resource_groups[each.value.resource_group_name].name

  virtual_network_name = azurerm_virtual_network.virtual_networks[each.value.virtual_network_name].name

  address_prefixes = each.value.address_prefixes
}
resource "azurerm_network_interface" "nics" {
  for_each            = var.virtual_machines
  name                = each.value.nic_name
  location            = azurerm_resource_group.resource_groups[each.value.resource_group_name].location
  resource_group_name = azurerm_resource_group.resource_groups[each.value.resource_group_name].name
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnets[each.value.subnet_name].id
    private_ip_address_allocation = "Dynamic"
  }
}
resource "azurerm_linux_virtual_machine" "vms" {

  for_each = var.virtual_machines

  name = each.value.name

  resource_group_name = azurerm_resource_group.resource_groups[each.value.resource_group_name].name

  location = azurerm_resource_group.resource_groups[each.value.resource_group_name].location

  size = each.value.size

  admin_username = each.value.admin_username

  admin_password = each.value.admin_password

  disable_password_authentication = false

  network_interface_ids = [

    azurerm_network_interface.nics[each.key].id
  ]

  os_disk {

    caching = "ReadWrite"

    storage_account_type = "Standard_LRS"
  }

  source_image_reference {

    publisher = "Canonical"

    offer = "0001-com-ubuntu-server-jammy"

    sku = "22_04-lts"

    version = "latest"
  }
}

*********************************************************************************************
