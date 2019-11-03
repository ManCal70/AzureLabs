# Azure Lab - Dual Juniper vSRX Active/Active + Azure Public LB

This lab will illustrate how to create an Azure Public load balancer, distribute traffic between two Juniper vSRX firewalls. The Azure public load balancer can be configured in two ways. 1) Default config - LB will translate the destination IP address to that of the BE pools VM (in this case vSRX), or 2)HA Ports config - This setting will NOT translate the incoming packets destination IP. This means the packets preserve its 5 tuples when being load balanced between the back end firewalls. I will show you how to configure both flavors of LB config.

### Design implications:
- When using the Azure Public LB default configuration, if you have multiple applications that are using the same destination port, you have to perform port translation. This is due to the fact that backed pool VMs are limited to one IP address. A NAT policy will need to be configured to perform the port translation. This can become cumbersome as you add more applications and create port translations. 

- Health check probes - There are 3 types of health probes you can use to check the health of the backend pool - TCP/HTTP/HTTPS - in this design, we are going to probe ssh to test firewalls responses. I will be enabling ssh service on the untrust interface where the probe will ingress. You should always secure the control plane by creating ACLs/Filters to only allow the required sources (that is beyond the scope of this document). Always keep in mind that probes are sent to the IP address of the firewalls, this means that when you are using the 'default' load balancer configuration, you may have conflics with firewall 'management' configurations (ssh etc...). 

- Floating IP configuration - Floating IP configuration will NOT perform destination NAT on the packets processed by the load balancer. The traffic will be load balanced and routed to the backend firewalls preserving the original 5 tuples. The firewall still requires a NAT rule which translates the destination IP address (Public load balancer IP address) to the private side resource. However, this configuration overcomes the multiple applications and port numbers limitations from the 'default' config (applications utilizing the same destination port). This style of configuration also mitigates the management traffic conflicts. 

# Default Config Topology Details
![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/Juniper_vSRX_HA_with_Public_LB/default-topo.png)


**Elements required**
<pre lang="...">
  - Resource Group 
  - Azure public Load balancer
    - Public IP address
    - Backed Pools
    - Health Probes
    - Load Balancing Rules
  - 2 x Juniper vSRX NVA Firewalls - Each with:
    - vNIC1 - Mapped to management subnet
    - vNIC2 - Mapped to UNTRUST subnet
    - vNIC3 - Mapped to TRUST subnet
    - Destination NAT policies - For incoming applications
    - Source NAT policies - For flow affinity to backe ends
    - TRUST & UNTRUST security zones
    - Custom routing instance (Type virtual-router)
    - Secutity policies 
  - 2 x VNETs
    - HUB VNET
      - IP Range - 10.0.0.0/16
      - Management Subnet - MGMT 10.0.254.0/24
      - UNTRUST Subnet - O-UNTRUST 10.0.100.0/24
      - TRUST Subnet - O-TRUST 10.0.99.0/24
    - Spoke VNET - 10.80.0.0/16
      - VM Workloads Subnet 10.80.0.99/24
    - Hub to Spoke VNET Peering
    - Ubuntu Virtual machine + Apache2
</pre>

**Create the Resource Group**
<pre lang="...">
az group create --name RG-PLB-TEST --location eastus
</pre>

**Create the Hub VNET**
<pre lang="...">
az group create --name HUB-VNET --resource-group TG-PLB-TEST --location eastus -address-prefix 10.0.0.0/16
</pre>

**Create the Subnets**
<pre lang="...">
az network vnet subnet create --vnet-name HUB-VNET --name MGMT --resource-group RG-PLB-TEST --address-prefixes 10.0.254.0/24
az network vnet subnet create --vnet-name HUB-VNET --name O-UNTRUST --resource-group RG-PLB-TEST --address-prefixes 10.0.0.0/24
az network vnet subnet create --vnet-name HUB-VNET --name O-TRUST --resource-group RG-PLB-TEST --address-prefixes 10.0.1.0/24
</pre>

**Create the Public IPs - When utilizing Public IPs with Standard SKU, an NSG is required on the Subnet/vNIC. Two public IPs will be created. 1) fxp0 - management interface 2) ge0 - UNTRUST/Interface facing interface **
<pre lang="...">
* fxp0 = Out of band management interface on vSRXs

az network public-ip create --name VSRX1-PIP-1 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
az network public-ip create --name VSRX1-PIP-2 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard

az network public-ip create --name VSRX2-PIP-1 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
az network public-ip create --name VSRX2-PIP-2 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
</pre>

**Create the vNICs**
<pre lang="...">
VSRX1
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX1-fxp0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX1-PIP-1 --private-ip-address 10.0.254.4

az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX1-ge0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX1-PIP-2 --private-ip-address 10.0.0.4

az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX1-ge1 --vnet-name HUB-VNET --subnet MGMT --private-ip-address 10.0.1.4

VSRX2
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX2-fxp0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX2-PIP-1 --private-ip-address 10.0.254.5

az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX2-ge0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX2-PIP-2 --private-ip-address 10.0.0.5

az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX2-ge1 --vnet-name HUB-VNET --subnet MGMT --private-ip-address 10.0.1.5
</pre>
