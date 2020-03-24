## Azure Networking Lab

In this lab, I will configure two firewalls (Juniper vSRX NVAs) each with a single vNIC. The two firewalls will be front ended by an Azure internal load balancer.

This design can be applied across multiple 3rd party NVA vendors that are able to support intra-zone security policies and enforcement.

## Dual NVA + single vNIC Firewalls Topology

<table><tr><td>
    <img src="https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_HA_SINGLE_NIC_%2B_LB/single-vnic-topo.png" lt="" title="Lab Topology" width="300" height="500"  />
</td></tr></table>


## Why single vNIC firewalls? 
When utilizing VMs/NVAs with multiple vNICs for ingress and egress (like firewalls), the use of source NAT is necessary to maintain flow symmetry/affinity. NAT can be an obstable with some applications which break when NATed, like Active Directory. A single vNIC NVA design, front ended with an Azure internal load balancer, provides the flow symmetry required when utilizing stateful firewalls.

## How to maintain flow symmetry/affinity with out source NAT?
The Azure load balancer hashing algorithm takes into account source IP/Port & destination IP/Port, and it is programmed in a way that is independent of the order of the fields. Means both flows/wings of the connection will maintain symmetry.

### Design
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

### Create vSRX PIPs for management
<pre lang= >
az network public-ip create --name VSRX1-E-PIP1 --allocation-method Static --resource-group RG-FW-LAB-E --location eastus --sku Standard
az network public-ip create --name VSRX2-E-PIP1 --allocation-method Static --resource-group RG-FW-LAB-E --location eastus --sku Standard

az network public-ip create --name VSRX1-W-PIP1 --allocation-method Static --resource-group RG-FW-LAB-W --location westus --sku Standard
az network public-ip create --name VSRX2-W-PIP1 --allocation-method Static --resource-group RG-FW-LAB-W --location westus --sku Standard
</pre>

### Creat vNICs for all VMS
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

### Create Contral Plane NSG
<pre lang= >
az network nsg create --resource-group RG-FW-LAB-W --name CPNSG-WEST --location westus
az network nsg create --resource-group RG-FW-LAB-E --name CPNSG-EAST --location eastus
</pre>

### Create CP NSG rule
<pre lang= >
WEST
az network nsg rule create -g RG-FW-LAB-W --nsg-name CPNSG-WEST -n ALLOW-SSH --priority 300 --source-address-prefixes Internet --destination-address-prefixes 10.0.254.0/24 --destination-port-ranges 22 --access Allow --protocol Tcp --description "Allow SSH to Management Subnet"
az network nsg rule create -g RG-FW-LAB-W --nsg-name CPNSG-WEST -n ALLOW-ICMP --priority 301 --source-address-prefixes Internet --destination-address-prefixes 10.0.254.0/24 --destination-port-ranges * --protocol Icmp --description "Allow ICMP to FW OOB interface"

EAST
az network nsg rule create -g RG-FW-LAB-E --nsg-name CPNSG-EAST -n ALLOW-SSH --priority 300 --source-address-prefixes Internet --destination-address-prefixes 10.10.254.0/24 --destination-port-ranges 22 --access Allow --protocol Tcp --description "Allow SSH to Management Subnet"
az network nsg rule create -g RG-FW-LAB-E --nsg-name CPNSG-EAST -n ALLOW-ICMP --priority 301 --source-address-prefixes Internet --destination-address-prefixes 10.10.254.0/24 --destination-port-ranges * --protocol Icmp --description "Allow ICMP to FW OOB interface"
</pre>

### Create Data plane NSG
<pre lang= >
WEST
az network nsg create --resource-group RG-FW-LAB-W --name DPNSG-WEST --location westus
az network nsg create --resource-group RG-FW-LAB-E --name DPNSG-EAST --location eastus

EAST
az network nsg rule create -g RG-FW-LAB-W --nsg-name DPNSG-WEST -n ALLOW-ALL --priority 300 --source-address-prefixes Internet --destination-address-prefixes * --destination-port-ranges * --access Allow --protocol * --description "Allow All"
az network nsg rule create -g RG-FW-LAB-E --nsg-name DPNSG-EAST -n ALLOW-ALL --priority 300 --source-address-prefixes Internet --destination-address-prefixes * --destination-port-ranges * --access Allow --protocol * --description "Allow All"
</pre>


### Associate vNICs with corresponding CP NSGs
<pre lang= >
az network nic update --resource-group RG-FW-LAB-W --name VSRX1-W-fxp0 --network-security-group CPNSG-WEST
az network nic update --resource-group RG-FW-LAB-W --name VSRX2-W-fxp0 --network-security-group CPNSG-WEST

az network nic update --resource-group RG-FW-LAB-E --name VSRX1-E-fxp0 --network-security-group CPNSG-EAST
az network nic update --resource-group RG-FW-LAB-E --name VSRX2-E-fxp0 --network-security-group CPNSG-EAST
</pre>

### Associate vNICs with corresponding DP NSGs
<pre lang= >
az network nic update --resource-group RG-FW-LAB-E --name VSRX1-E-ge0 --network-security-group DPNSG-EAST
az network nic update --resource-group RG-FW-LAB-E --name VSRX2-E-ge0 --network-security-group DPNSG-EAST

az network nic update --resource-group RG-FW-LAB-W --name VSRX1-W-ge0 --network-security-group DPNSG-WEST
az network nic update --resource-group RG-FW-LAB-W --name VSRX2-W-ge0 --network-security-group DPNSG-WEST
</pre>

### Create Firewalls VSRXs
<pre lang= >
WEST
az vm create --resource-group RG-FW-LAB-W --location westus --name VSRX1-W --size Standard_DS3_v2 --nics VSRX1-W-fxp0 VSRX1-W-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait
az vm create --resource-group RG-FW-LAB-W --location westus --name VSRX2-W --size Standard_DS3_v2 --nics VSRX2-W-fxp0 VSRX2-W-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait

EAST
az vm create --resource-group RG-FW-LAB-E --location eastus --name VSRX1-E --size Standard_DS3_v2 --nics VSRX1-E-fxp0 VSRX1-E-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait
az vm create --resource-group RG-FW-LAB-E --location eastus --name VSRX2-E --size Standard_DS3_v2 --nics VSRX2-E-fxp0 VSRX2-E-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait
</pre>

### create ILB
<pre lang= >
az network lb create --resource-group RG-FW-LAB-W --name ILB-W --frontend-ip-name ILBFE-W --private-ip-address 10.0.0.254 --backend-pool-name ILBBE-W --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --location westus --sku Standard
az network lb create --resource-group RG-FW-LAB-E --name ILB-E --frontend-ip-name ILBFE-E --private-ip-address 10.10.0.254 --backend-pool-name ILBBE-E --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --location eastus --sku Standard
</pre>

### Add vNIC to LB
<pre lang= >
WEST
az network nic ip-config update -g RG-FW-LAB-W --nic-name VSRX1-W-ge0 -n ipconfig1 --lb-address-pool ILBBE-W --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --lb-name ILB-W
az network nic ip-config update -g RG-FW-LAB-W --nic-name VSRX2-W-ge0 -n ipconfig1 --lb-address-pool ILBBE-W --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --lb-name ILB-W

EAST
az network nic ip-config update -g RG-FW-LAB-E --nic-name VSRX1-E-ge0 -n ipconfig1 --lb-address-pool ILBBE-E --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --lb-name ILB-E
az network nic ip-config update -g RG-FW-LAB-E --nic-name VSRX2-E-ge0 -n ipconfig1 --lb-address-pool ILBBE-E --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --lb-name ILB-E
</pre>

### Create probe
<pre lang= >
az network lb probe create --lb-name ILB-W --name FWPROBE-W --port 22 --protocol tcp --resource-group  RG-FW-LAB-W
az network lb probe create --lb-name ILB-E --name FWPROBE-E --port 22 --protocol tcp --resource-group  RG-FW-LAB-E
</pre>

### Create HA ports (route all traffc and ports to backe end) lb rule
<pre lang= >
az network lb rule create -g RG-FW-LAB-W --lb-name ILB-W --name LBRULE-W  --protocol All --frontend-port 0 --backend-port 0 --frontend-ip-name  ILBFE-W --backend-pool-name ILBBE-W --probe-name FWPROBE-W
az network lb rule create -g RG-FW-LAB-E --lb-name ILB-E --name LBRULE-E  --protocol All --frontend-port 0 --backend-port 0 --frontend-ip-name  ILBFE-E --backend-pool-name ILBBE-E --probe-name FWPROBE-E
</pre>

### UDR route table
<pre lang= >
az network route-table create --name RT-2-LB-W -g RG-FW-LAB-W --location westus --disable-bgp-route-propagation true
az network route-table create --name RT-2-LB-E -g RG-FW-LAB-E --location eastus --disable-bgp-route-propagation true
</pre>

### Create Routes
<pre lang= >
az network route-table route create --name RT-2-LB-E -g RG-FW-LAB-E --route-table-name RT-2-LB-E --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.10.0.254
az network route-table route create --name RT-2-LB-W -g RG-FW-LAB-W --route-table-name RT-2-LB-W --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.0.254
</pre>

### Apply UDR to the Spoke VNETS
<pre lang= >
az network vnet subnet update --vnet-name SPK1-WEST --name SPK1-WEST-SUB --resource-group RG-FW-LAB-W --route-table RT-2-LB-W
az network vnet subnet update --vnet-name SPK2-WEST --name SPK2-WEST-SUB --resource-group RG-FW-LAB-W --route-table RT-2-LB-W

az network vnet subnet update --vnet-name SPK1-EAST --name SPK1-EAST-SUB --resource-group RG-FW-LAB-E --route-table RT-2-LB-E
az network vnet subnet update --vnet-name SPK2-EAST --name SPK2-EAST-SUB --resource-group RG-FW-LAB-E --route-table RT-2-LB-E
</pre>

### view the UDR
<pre lang= >
az network route-table route show -g RG-FW-LAB-W --name RT-2-LB-W --route-table-name RT-2-LB-W --output table
az network route-table route show -g RG-FW-LAB-E --name RT-2-LB-E --route-table-name RT-2-LB-E --output table
</pre>


### NVA Firewall Configuration
<pre lang= >
<b>Interface configuration</b>
set interfaces ge-0/0/0 description "Firewal vNIC"
set interfaces ge-0/0/0 unit 0 family inet dhcp
set interfaces ge-0/0/1 disable
set interfaces fxp0 unit 0

<b>Configure security zone, spoke address prefixes</b>
set security zones security-zone TRUST address-book address 10.11.0.0/24 10.11.0.0/24 <b>>>> Spoke Subnet</b>
set security zones security-zone TRUST address-book address 10.12.0.0/24 10.12.0.0/24 <b>>>> Spoke Subnet</b>
set security zones security-zone TRUST host-inbound-traffic system-services all
set security zones security-zone TRUST host-inbound-traffic protocols all
set security zones security-zone TRUST interfaces ge-0/0/0.0

<b>Security Policies to allow Spokes to communicate with each other:</b>
set security policies from-zone TRUST to-zone TRUST policy 11-TO-12 match source-address 10.11.0.0/24
set security policies from-zone TRUST to-zone TRUST policy 11-TO-12 match destination-address 10.12.0.0/24
set security policies from-zone TRUST to-zone TRUST policy 11-TO-12 match application any
set security policies from-zone TRUST to-zone TRUST policy 11-TO-12 then permit

set security policies from-zone TRUST to-zone TRUST policy 12-TO-11 match source-address 10.12.0.0/24
set security policies from-zone TRUST to-zone TRUST policy 12-TO-11 match destination-address 10.11.0.0/24
set security policies from-zone TRUST to-zone TRUST policy 12-TO-11 match application any
set security policies from-zone TRUST to-zone TRUST policy 12-TO-11 then permit

<b>Single routing instance config</b>
set routing-instances VR1 instance-type virtual-router
set routing-instances VR1 routing-options static route 168.63.129.16/32 next-hop 10.10.0.1 <b>>>> Probe route</b>
set routing-instances VR1 routing-options static route 0.0.0.0/0 next-hop 10.10.0.1 <b>>>> Default to fabric</b>
set routing-instances VR1 interface ge-0/0/0.0
</pre>
