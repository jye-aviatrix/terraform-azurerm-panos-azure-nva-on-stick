# terraform-azurerm-panos-azure-nva

This Repo will create Palo Alto Networks VM-Series firewalls in Azure, it will also bootstrap the Firewall, as well as provision Azure Internal Load Balancer.

- All resources will be placed under a Resource Group. A virtual network with mgmt, trust subnet will be created.
- It will also bootstrap the Palo with preconfigured policies to allow trust -> trust, deny everything else.
- It will create an internal load balanacer, and register firewall as it's backend.

![Topology](https://raw.githubusercontent.com/jye-aviatrix/terraform-azurerm-panos-azure-nva-on-stick/master/az-vnet-hub-with-palo-fw-on-stick-behind-lb.png)

mgmt subnet will be associated with DefaultNSG, where it allows incoming connection your egress public IP for management, it also allow inocming connection within vNet CIDR, but block everything else.

Palo will have three interfaces
- eth0 -> mgmt
- eth1 -> trust

Trust have IP Forwarding enabled. mgmt have IP Forwarding disabled.

Static route will be configured
- 0/0 -> first IP of trust subnet via eth1
- 168.63.129.16/32 -> first IP of trust subnet via eth1


HTTPs enabled for trust interface management, this will be used for Health Check from Internal Network Load Balancer.

Included bootstrapped firewall username: ```fwadmin```
Password: ```sb%BSu/.T+j3```

It's reccommended you create your own bootstrap.xml, so that it will have different default password and SSH public Key value

https://docs.paloaltonetworks.com/vm-series/10-1/vm-series-deployment/bootstrap-the-vm-series-firewall/create-the-bootstrapxml-file

Reference to bootstrap/bootstrap.xml, find and replace VM name and static routes target IP wth following:
```
    ${palo_vm_name} -> Two references
    ${trust_subnet_router} -> First IP of trust subnet, three references for the three RFC1918 ranges
```
## proiders.tf
```
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
    }
    http = {
      source = "hashicorp/http"
    }
    random = {
      source = "hashicorp/random"
    }
    local = {
      source = "hashicorp/local"
    }
  }
}

# Configure the Microsoft Azure Provider
provider "azurerm" {
  features {}
}
```

## Tested environment
```
Terraform v1.3.7
on linux_amd64
+ provider registry.terraform.io/hashicorp/azurerm v3.45.0
+ provider registry.terraform.io/hashicorp/http v3.2.1
+ provider registry.terraform.io/hashicorp/local v2.3.0
+ provider registry.terraform.io/hashicorp/random v3.4.3
```

## Cost estimate
```
 Name                                                                Monthly Qty  Unit                      Monthly Cost

 azurerm_lb.this
 └─ Data processed                                            Monthly cost depends on usage: $0.00 per GB

 azurerm_lb_rule.this
 └─ Rule usage                                                               730  hours                            $0.00

 azurerm_storage_account.palo_bootstrap
 ├─ Capacity                                                  Monthly cost depends on usage: $0.0208 per GB
 ├─ List and create container operations                      Monthly cost depends on usage: $0.05 per 10k operations
 ├─ Read operations                                           Monthly cost depends on usage: $0.004 per 10k operations
 ├─ All other operations                                      Monthly cost depends on usage: $0.004 per 10k operations
 └─ Blob index                                                Monthly cost depends on usage: $0.03 per 10k tags

 module.palo_byol[0].azurerm_linux_virtual_machine.palo_byol
 ├─ Instance usage (pay as you go, Standard_D3_v2)                           730  hours                          $213.89
 └─ os_disk
    ├─ Storage (E4)                                                            1  months                           $2.40
    └─ Disk operations                                        Monthly cost depends on usage: $0.002 per 10k operations

 module.palo_byol[0].azurerm_public_ip.mgmt_pip
 └─ IP address (static)                                                      730  hours                            $3.65


 module.palo_byol[1].azurerm_linux_virtual_machine.palo_byol
 ├─ Instance usage (pay as you go, Standard_D3_v2)                           730  hours                          $213.89
 └─ os_disk
    ├─ Storage (E4)                                                            1  months                           $2.40
    └─ Disk operations                                        Monthly cost depends on usage: $0.002 per 10k operations

 module.palo_byol[1].azurerm_public_ip.mgmt_pip
 └─ IP address (static)                                                      730  hours                            $3.65

 OVERALL TOTAL                                                                                                   $439.88
──────────────────────────────────
40 cloud resources were detected:
∙ 9 were estimated, 4 of which include usage-based costs, see https://infracost.io/usage-file
∙ 17 were free, rerun with --show-skipped to see details
∙ 14 are not supported yet, rerun with --show-skipped to see details
```