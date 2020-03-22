## Azure Networking Lab

In this lab, I will configure two firewalls (Juniper vSRX NVAs) each with a single vNIC. The two firewalls will be front ended by an Azure internal load balancer.

## Why single vNIC firewalls? 
When utilizing VMs/NVAs with multiple vNICs for ingress and egress (like firewalls), the use of source NAT is necessary to maintain flow symmetry/affinity. This becomes a obstable with some applications that break when NATed, like Active Directory. A single vNIC design removes this obstacle. 

## How to maintain flow symmetry/affinity with out source NAT?
The Azure load balancer hashing algorithm takes into account source IP/Port & destination IP/Port, and it is programmed in a way that is independent of the order of the fields. Means both flows/wings of the connection will maintain symmetry.

<p align="left">
<b>Design</left></b>
<pre lang= >
<b>* 2 x single vNIC Network Virtual Appliances (NVA) Firewalls (Juniper vSRX)</b>
<b>* Active/Active design front ended by an Internal LB.</b>
<b>* 2 regions with a Hub and two spokes which are VNET peered</b>
<b>* 2 hubs are Global VNET peered</b>
</pre>

<p align="left">
<b>Topology</center></b>

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_HA_SINGLE_NIC_%2B_LB/topology.png)</kbd>
<p align="center">


<p align="left"><b>This design can be applied across multiple 3rd party NVA vendors that are able to support intra-zone security policies and enforcement</p></b>
</p>

### Create two resource groups - to separate East and West region elements
<pre lang= >
az group create --name RG-FW-LAB-E --location eastus --output table
az group create --name RG-FW-LAB-W --location westus --output table
</pre>

### Create storage account for bootdiagnostics (not always required)
<pre lang= >
az storage account create -n mcbootdiag -g RG-FW-LAB-E -l eastus --sku Standard_LRS
</pre>

### Create VNETs
<pre lang= >
WEST
az network vnet create --name HUB-WEST --resource-group RG-FW-LAB-W --location westus --address-prefix 10.0.0.0/16
az network vnet create --name SPK1-WEST --resource-group RG-FW-LAB-W --location westus --address-prefix 10.1.0.0/16
az network vnet create --name SPK2-WEST --resource-group RG-FW-LAB-W --location westus --address-prefix 10.2.0.0/16
</pre>
<pre lang= >
EAST
az network vnet create --name HUB-EAST --resource-group RG-FW-LAB-E --location eastus --address-prefix 10.10.0.0/16
az network vnet create --name SPK1-EAST --resource-group RG-FW-LAB-E --location eastus --address-prefix 10.11.0.0/16
az network vnet create --name SPK2-EAST --resource-group RG-FW-LAB-E --location eastus --address-prefix 10.12.0.0/16
</pre>
### Create Subnets
<pre lang= >
WEST
az network vnet subnet create --vnet-name HUB-WEST --name MGT-WEST-SUB --resource-group RG-FW-LAB-W --address-prefixes 10.0.254.0/24 --output table
az network vnet subnet create --vnet-name HUB-WEST --name FWSUB-WEST-SUB --resource-group RG-FW-LAB-W --address-prefixes 10.0.0.0/24 --output table
az network vnet subnet create --vnet-name SPK1-WEST --name SPK1-WEST-SUB --resource-group RG-FW-LAB-W --address-prefixes 10.1.0.0/24 --output table
az network vnet subnet create --vnet-name SPK2-WEST --name SPK2-WEST-SUB --resource-group RG-FW-LAB-W --address-prefixes 10.2.0.0/24 --output table
</pre>
<pre lang= >
EAST
az network vnet subnet create --vnet-name HUB-EAST --name MGT-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.10.254.0/24 --output table
az network vnet subnet create --vnet-name HUB-EAST --name FWSUB-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.10.0.0/24 --output table
az network vnet subnet create --vnet-name SPK1-EAST --name SPK1-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.11.0.0/24 --output table
az network vnet subnet create --vnet-name SPK2-EAST --name SPK2-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.12.0.0/24 --output table
</pre>

Create vSRX PIPs for management"
<pre lang= >
az network public-ip create --name VSRX1-E-PIP1 --allocation-method Static --resource-group RG-FW-LAB-E --location eastus --sku Standard
az network public-ip create --name VSRX2-E-PIP1 --allocation-method Static --resource-group RG-FW-LAB-E --location eastus --sku Standard

az network public-ip create --name VSRX1-W-PIP1 --allocation-method Static --resource-group RG-FW-LAB-W --location westus --sku Standard
az network public-ip create --name VSRX2-W-PIP1 --allocation-method Static --resource-group RG-FW-LAB-W --location westus --sku Standard
</pre>

Creat vNICs for all VMS
<pre lang= >
WEST
az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX1-W-fxp0 --vnet-name HUB-WEST --subnet MGT-WEST-SUB --public-ip-address  VSRX1-W-PIP1 --private-ip-address 10.0.254.4
az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX2-W-fxp0 --vnet-name HUB-WEST --subnet MGT-WEST-SUB --public-ip-address  VSRX2-W-PIP1 --private-ip-address 10.0.254.5

az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX1-W-ge0 --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --private-ip-address 10.0.0.4 --ip-forwarding
az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX2-W-ge0 --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --private-ip-address 10.0.0.5 --ip-forwarding

az network nic create --resource-group RG-FW-LAB-W --location westus --name VM1-SPK1-W --vnet-name SPK1-WEST --subnet SPK1-WEST-SUB --private-ip-address 10.1.0.4
az network nic create --resource-group RG-FW-LAB-W --location westus --name VM1-SPK2-W --vnet-name SPK2-WEST --subnet SPK2-WEST-SUB --private-ip-address 10.2.0.4

EAST
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX1-E-fxp0 --vnet-name HUB-EAST --subnet MGT-EAST-SUB --public-ip-address  VSRX1-E-PIP1 --private-ip-address 10.10.254.4
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX2-E-fxp0 --vnet-name HUB-EAST --subnet MGT-EAST-SUB --public-ip-address  VSRX2-E-PIP1 --private-ip-address 10.10.254.5

az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX1-E-ge0 --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --private-ip-address 10.10.0.4 --ip-forwarding
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX2-E-ge0 --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --private-ip-address 10.10.0.5 --ip-forwarding

az network nic create --resource-group RG-FW-LAB-E --location eastus --name VM1-SPK1-E --vnet-name SPK1-EAST --subnet SPK1-EAST-SUB --private-ip-address 10.11.0.4
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VM1-SPK2-E --vnet-name SPK2-EAST --subnet SPK2-EAST-SUB --private-ip-address 10.12.0.4
</pre>


### Key details