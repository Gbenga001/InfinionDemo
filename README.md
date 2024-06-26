# InfinionDemo
Create an externally accessible static website using Azure Virtual Machine
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.0.0"
    }
  }
}

provider "azurerm" {
    features {}
}

# Create a resource group
resource "azurerm_resource_group" "static_website_rg" {
  name     = "static-website-rg"
  location = "West Europe"
}

# Create virtual network
resource "azurerm_virtual_network" "static_website_vnet" {
  name                = "static-website-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.static_website_rg.location
  resource_group_name = azurerm_resource_group.static_website_rg.name
}

# Create subnet
resource "azurerm_subnet" "static_website_subnet" {
  name                 = "static-website-subnet"
  resource_group_name  = azurerm_resource_group.static_website_rg.name
  virtual_network_name = azurerm_virtual_network.static_website_vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Create public IPs
resource "azurerm_public_ip" "static_website_public_ip" {
  name                = "static-website-public-ip"
  location            = azurerm_resource_group.static_website_rg.location
  resource_group_name = azurerm_resource_group.static_website_rg.name
  allocation_method   = "Static"
}

# Output the public IPs
output "public_ip_address" {
  value = azurerm_public_ip.static_website_public_ip.ip_address
}

# Network Security Group
resource "azurerm_network_security_group" "static_website_nsg" {
  name                = "static-website-nsg"
  location            = azurerm_resource_group.static_website_rg.location
  resource_group_name = azurerm_resource_group.static_website_rg.name

  security_rule {
    name = "allow_http"
    access = "Allow"
    priority = 100
    direction = "Inbound"
    destination_port_range = "80"
    protocol = "Tcp"
    source_port_range = "*"
    source_address_prefix = "*"
    destination_address_prefix = "*"
  }
}

# Network Interface
resource "azurerm_network_interface" "static_website_nic" {
  name                = "static-website-nic"
  location            = azurerm_resource_group.static_website_rg.location
  resource_group_name = azurerm_resource_group.static_website_rg.name
  ip_configuration {
    name                          = "internal"
    subnet_id                    = azurerm_subnet.static_website_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.static_website_public_ip.id
  }
}

# Virtual Machine


# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "static_website_nisga" {
  network_interface_id      = azurerm_network_interface.static_website_nic.id
  network_security_group_id = azurerm_network_security_group.static_website_nsg.id
}

resource "azurerm_linux_virtual_machine" "static_website_vm" {
  name                = "static-website-vm"
  resource_group_name = azurerm_resource_group.static_website_rg.name
  location            = azurerm_resource_group.static_website_rg.location
  size                = "Standard_F2"
  admin_username      = "adminuser"
  network_interface_ids = [
    azurerm_network_interface.static_website_nic.id,
  ]

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}
