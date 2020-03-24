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

EAST
az network vnet subnet create --vnet-name HUB-EAST --name MGT-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.10.254.0/24 --output table
az network vnet subnet create --vnet-name HUB-EAST --name FWSUB-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.10.0.0/24 --output table
az network vnet subnet create --vnet-name SPK1-EAST --name SPK1-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.11.0.0/24 --output table
az network vnet subnet create --vnet-name SPK2-EAST --name SPK2-EAST-SUB --resource-group RG-FW-LAB-E --address-prefixes 10.12.0.0/24 --output table
</pre>

### Create hub to spoke VNET peerings
<pre lang= >
az network vnet peering create -g RG-FW-LAB-W --name HUB-W-SPK1 --vnet-name HUB-WEST --remote-vnet SPK1-WEST --allow-forwarded-traffic --allow-vnet-access --output table
az network vnet peering create -g RG-FW-LAB-W --name SPK1-HUB-W --vnet-name SPK1-WEST --remote-vnet HUB-WEST --allow-forwarded-traffic --allow-vnet-access --output table

az network vnet peering create -g RG-FW-LAB-W --name HUB-W-SPK2 --vnet-name HUB-WEST --remote-vnet SPK2-WEST --allow-forwarded-traffic --allow-vnet-access --output table
az network vnet peering create -g RG-FW-LAB-W --name SPK2-HUB-W --vnet-name SPK2-WEST --remote-vnet HUB-WEST --allow-forwarded-traffic --allow-vnet-access --output table

az network vnet peering create -g RG-FW-LAB-E --name HUB-E-SPK1 --vnet-name HUB-EAST --remote-vnet SPK1-EAST --allow-forwarded-traffic --allow-vnet-access --output table
az network vnet peering create -g RG-FW-LAB-E --name SPK1-E-HUB --vnet-name SPK1-EAST --remote-vnet HUB-EAST --allow-forwarded-traffic --allow-vnet-access --output table

az network vnet peering create -g RG-FW-LAB-E  --name HUB-E-SPK2 --vnet-name HUB-EAST --remote-vnet SPK2-EAST --allow-forwarded-traffic --allow-vnet-access --output table
az network vnet peering create -g RG-FW-LAB-E  --name SPK2-E-HUB --vnet-name SPK2-EAST --remote-vnet HUB-EAST --allow-forwarded-traffic --allow-vnet-access --output table
</pre>

### Create vSRX PIPs for management
<pre lang= >
az network public-ip create --name VSRX1-E-PIP1 --allocation-method Static --resource-group RG-FW-LAB-E --location eastus --sku Standard
az network public-ip create --name VSRX2-E-PIP1 --allocation-method Static --resource-group RG-FW-LAB-E --location eastus --sku Standard

az network public-ip create --name VSRX1-W-PIP1 --allocation-method Static --resource-group RG-FW-LAB-W --location westus --sku Standard
az network public-ip create --name VSRX2-W-PIP1 --allocation-method Static --resource-group RG-FW-LAB-W --location westus --sku Standard
</pre>

### Create vNICs for all VMS
<pre lang= >
WEST
az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX1-W-fxp0 --vnet-name HUB-WEST --subnet MGT-WEST-SUB --public-ip-address  VSRX1-W-PIP1 --private-ip-address 10.0.254.4
az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX2-W-fxp0 --vnet-name HUB-WEST --subnet MGT-WEST-SUB --public-ip-address  VSRX2-W-PIP1 --private-ip-address 10.0.254.5

az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX1-W-ge0 --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --private-ip-address 10.0.0.4 --ip-forwarding
az network nic create --resource-group RG-FW-LAB-W --location westus --name VSRX2-W-ge0 --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --private-ip-address 10.0.0.5 --ip-forwarding

West Test VM vNICs
az network nic create --resource-group RG-FW-LAB-W --location westus --name VM1-SPK1-W-eth0 --vnet-name SPK1-WEST --subnet SPK1-WEST-SUB --private-ip-address 10.1.0.4
az network nic create --resource-group RG-FW-LAB-W --location westus --name VM1-SPK2-W-eth0 --vnet-name SPK2-WEST --subnet SPK2-WEST-SUB --private-ip-address 10.2.0.4

EAST
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX1-E-fxp0 --vnet-name HUB-EAST --subnet MGT-EAST-SUB --public-ip-address  VSRX1-E-PIP1 --private-ip-address 10.10.254.4
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX2-E-fxp0 --vnet-name HUB-EAST --subnet MGT-EAST-SUB --public-ip-address  VSRX2-E-PIP1 --private-ip-address 10.10.254.5

az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX1-E-ge0 --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --private-ip-address 10.10.0.4 --ip-forwarding
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VSRX2-E-ge0 --vnet-name HUB-EAST --subnet FWSUB-EAST-SUB --private-ip-address 10.10.0.5 --ip-forwarding

East Test VM vNICs
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VM1-SPK1-E-eth0 --vnet-name SPK1-EAST --subnet SPK1-EAST-SUB --private-ip-address 10.11.0.4
az network nic create --resource-group RG-FW-LAB-E --location eastus --name VM1-SPK2-E-eth0 --vnet-name SPK2-EAST --subnet SPK2-EAST-SUB --private-ip-address 10.12.0.4
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

### Create firewall Data plane NSG
<pre lang= >
WEST
az network nsg create --resource-group RG-FW-LAB-W --name DPNSG-WEST --location westus

EAST
az network nsg create --resource-group RG-FW-LAB-E --name DPNSG-EAST --location eastus
</pre>

### Create firewall data plane NSG rule
<pre lang= >
az network nsg rule create -g RG-FW-LAB-W --nsg-name DPNSG-WEST -n ALLOW-ALL-IN --priority 300 --source-address-prefixes * --destination-address-prefixes * --destination-port-ranges * --access Allow --protocol * --description "Allow All" --direction Inbound
az network nsg rule create -g RG-FW-LAB-E --nsg-name DPNSG-EAST -n ALLOW-ALL-IN --priority 300 --source-address-prefixes * --destination-address-prefixes * --destination-port-ranges * --access Allow --protocol * --description "Allow All" --direction Inbound

az network nsg rule create -g RG-FW-LAB-W --nsg-name DPNSG-WEST -n ALLOW-ALL-OUT --priority 300 --source-address-prefixes * --destination-address-prefixes * --destination-port-ranges * --access Allow --protocol * --description "Allow All" --direction Outbound
az network nsg rule create -g RG-FW-LAB-E --nsg-name DPNSG-EAST -n ALLOW-ALL-OUT --priority 300 --source-address-prefixes * --destination-address-prefixes * --destination-port-ranges * --access Allow --protocol * --description "Allow All" --direction Outbound

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

### Create firewall and test VMs
<pre lang= >
WEST firewalls
az vm create --resource-group RG-FW-LAB-W --location westus --name VSRX1-W --size Standard_DS3_v2 --nics VSRX1-W-fxp0 VSRX1-W-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait
az vm create --resource-group RG-FW-LAB-W --location westus --name VSRX2-W --size Standard_DS3_v2 --nics VSRX2-W-fxp0 VSRX2-W-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait

Test VMS
az vm create -n W-SPK1-VM -g RG-FW-LAB-W --image UbuntuLTS --admin-username lab-user --admin-password AzLabPass1234 --nics VM1-SPK1-W-eth0 --boot-diagnostics-storage mcbootdiag --no-wait
az vm create -n W-SPK2-VM -g RG-FW-LAB-W --image UbuntuLTS --admin-username lab-user --admin-password AzLabPass1234 --nics VM1-SPK2-W-eth0 --boot-diagnostics-storage mcbootdiag --no-wait


EAST firewalls
az vm create --resource-group RG-FW-LAB-E --location eastus --name VSRX1-E --size Standard_DS3_v2 --nics VSRX1-E-fxp0 VSRX1-E-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait
az vm create --resource-group RG-FW-LAB-E --location eastus --name VSRX2-E --size Standard_DS3_v2 --nics VSRX2-E-fxp0 VSRX2-E-ge0 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234 --boot-diagnostics-storage mcbootdiag --no-wait

Test VMs
az vm create -n E-SPK1-VM -g RG-FW-LAB-E --image UbuntuLTS --admin-username lab-user --admin-password AzLabPass1234 --nics VM1-SPK1-E-eth0 --boot-diagnostics-storage mcbootdiag --no-wait
az vm create -n E-SPK2-VM -g RG-FW-LAB-E --image UbuntuLTS --admin-username lab-user --admin-password AzLabPass1234 --nics VM1-SPK2-E-eth0 --boot-diagnostics-storage mcbootdiag --no-wait


<b>Once the VM is up and running, run the following to update and install apache2:</b>
1- sudo apt update
2- sudo apt upgrade -y
3- sudo apt install apache2 -y

</pre>

### Create ILB
<pre lang= >
WEST
az network lb create --resource-group RG-FW-LAB-W --name ILB-W --frontend-ip-name ILBFE-W --private-ip-address 10.0.0.254 --backend-pool-name ILBBE-W --vnet-name HUB-WEST --subnet FWSUB-WEST-SUB --location westus --sku Standard

EAST
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
WEST
az network lb probe create --lb-name ILB-W --name FWPROBE-W --port 22 --protocol tcp --resource-group  RG-FW-LAB-W

EAST
az network lb probe create --lb-name ILB-E --name FWPROBE-E --port 22 --protocol tcp --resource-group  RG-FW-LAB-E
</pre>

### Create HA ports (route all traffc and ports to backe end) lb rule
<pre lang= >
WEST
az network lb rule create -g RG-FW-LAB-W --lb-name ILB-W --name LBRULE-W  --protocol All --frontend-port 0 --backend-port 0 --frontend-ip-name  ILBFE-W --backend-pool-name ILBBE-W --probe-name FWPROBE-W


EAST
az network lb rule create -g RG-FW-LAB-E --lb-name ILB-E --name LBRULE-E  --protocol All --frontend-port 0 --backend-port 0 --frontend-ip-name  ILBFE-E --backend-pool-name ILBBE-E --probe-name FWPROBE-E
</pre>

### UDR route table
<pre lang= >
WEST
az network route-table create --name RT-2-LB-W -g RG-FW-LAB-W --location westus --disable-bgp-route-propagation true

EAST
az network route-table create --name RT-2-LB-E -g RG-FW-LAB-E --location eastus --disable-bgp-route-propagation true
</pre>

### Create Routes
<pre lang= >
EAST
az network route-table route create --name RT-2-LB-E -g RG-FW-LAB-E --route-table-name RT-2-LB-E --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.10.0.254

WEST
az network route-table route create --name RT-2-LB-W -g RG-FW-LAB-W --route-table-name RT-2-LB-W --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.0.254
</pre>

### Apply UDR to the Spoke VNETS
<pre lang= >
WEST
az network vnet subnet update --vnet-name SPK1-WEST --name SPK1-WEST-SUB --resource-group RG-FW-LAB-W --route-table RT-2-LB-W
az network vnet subnet update --vnet-name SPK2-WEST --name SPK2-WEST-SUB --resource-group RG-FW-LAB-W --route-table RT-2-LB-W

EAST
az network vnet subnet update --vnet-name SPK1-EAST --name SPK1-EAST-SUB --resource-group RG-FW-LAB-E --route-table RT-2-LB-E
az network vnet subnet update --vnet-name SPK2-EAST --name SPK2-EAST-SUB --resource-group RG-FW-LAB-E --route-table RT-2-LB-E
</pre>

### view the UDR
<pre lang= >
WEST
az network route-table route show -g RG-FW-LAB-W --name RT-2-LB-W --route-table-name RT-2-LB-W --output table

EAST
az network route-table route show -g RG-FW-LAB-E --name RT-2-LB-E --route-table-name RT-2-LB-E --output table
</pre>


### NVA Firewall Configuration
<pre lang= >
<b>EAST Firewall config:</b>
<b>First command you should run in firewall:</b>

delete security

set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 match source-address 10.11.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 match destination-address 10.12.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 match application any
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 then permit
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 then log session-init
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 then log session-close

set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 match source-address 10.12.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 match destination-address 10.11.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 match application any
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 then permit
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 then log session-init
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 then log session-close

set security zones security-zone TRUST address-book address 10.11.0.0/24 10.11.0.0/24
set security zones security-zone TRUST address-book address 10.12.0.0/24 10.12.0.0/24
set security zones security-zone TRUST host-inbound-traffic system-services all
set security zones security-zone TRUST host-inbound-traffic protocols all
set security zones security-zone TRUST interfaces ge-0/0/0.0

set interfaces ge-0/0/0 description VNETSUB
set interfaces ge-0/0/0 unit 0 family inet dhcp
set interfaces ge-0/0/1 disable
set interfaces fxp0 unit 0

set routing-instances VR1 instance-type virtual-router
set routing-instances VR1 routing-options static route 168.63.129.16/32 next-hop 10.10.0.1
set routing-instances VR1 routing-options static route 0.0.0.0/0 next-hop 10.10.0.1
set routing-instances VR1 interface ge-0/0/0.0

<b>WEST Firewall Config:</b>
<b>Same as east firewalls:</b>

delete security

set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 match source-address 10.1.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 match destination-address 10.2.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 match application any
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 then permit
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 then log session-init
set security policies from-zone TRUST to-zone TRUST policy SPK1-TO-SPOK2 then log session-close

set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 match source-address 10.2.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 match destination-address 10.1.0.0/24
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 match application any
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 then permit
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 then log session-init
set security policies from-zone TRUST to-zone TRUST policy SPK2-TO-SPOK1 then log session-close

set security zones security-zone TRUST address-book address 10.1.0.0/24 10.1.0.0/24
set security zones security-zone TRUST address-book address 10.2.0.0/24 10.2.0.0/24
set security zones security-zone TRUST host-inbound-traffic system-services all
set security zones security-zone TRUST host-inbound-traffic protocols all
set security zones security-zone TRUST interfaces ge-0/0/0.0
set interfaces ge-0/0/0 description VNETSUB
set interfaces ge-0/0/0 unit 0 family inet dhcp
set interfaces ge-0/0/1 disable
set interfaces fxp0 unit 0
set routing-instances VR1 instance-type virtual-router
set routing-instances VR1 routing-options static route 168.63.129.16/32 next-hop 10.0.0.1
set routing-instances VR1 routing-options static route 0.0.0.0/0 next-hop 10.0.0.1
set routing-instances VR1 interface ge-0/0/0.0
</pre>

### As you can see in the firewall config, we have two "intra-zone" policies. Each security policy allows specific VNET to VNET traffic. 

### Here are some verification commands
#### Health probe in firewall session table
<pre lang= >
lab-user@VSRX1-E# run show security flow session
Session ID: 29016, Policy name: self-traffic-policy/1, Timeout: 1792, Valid
  In: <b>168.63.129.16/62474</b> --> 10.10.0.4/22;tcp, Conn Tag: 0x0, If: <b>ge-0/0/0.0</b>, Pkts: 3, Bytes: 132,
  Out: 10.10.0.4/22 --> 168.63.129.16/62474;tcp, Conn Tag: 0x0, If: .local..6, Pkts: 2, Bytes: 112,
Total sessions: 1
</pre>

### View route tables and routes (UDRs) created
<pre lang= >
az network route-table route show -g RG-FW-LAB-E  --route-table-name RT-2-LB-E --name TO-SPK1-E --output table

AddressPrefix    Name       NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ---------  ------------------  ----------------  -------------------  ---------------
10.11.0.0/24     TO-SPK1-E  10.10.0.254         VirtualAppliance  Succeeded            RG-FW-LAB-E


az network route-table route show -g RG-FW-LAB-E  --route-table-name RT-2-LB-E --name TO-SPK2-E --output table

AddressPrefix    Name       NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ---------  ------------------  ----------------  -------------------  ---------------
10.12.0.0/24     TO-SPK2-E  10.10.0.254         VirtualAppliance  Succeeded            RG-FW-LAB-E

az network route-table route show -g RG-FW-LAB-W  --route-table-name RT-2-LB-W --name TO-SPK1-W --output table

AddressPrefix    Name       NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ---------  ------------------  ----------------  -------------------  ---------------
10.1.0.0/24      TO-SPK1-W  10.0.0.254          VirtualAppliance  Succeeded            RG-FW-LAB-W

az network route-table route show -g RG-FW-LAB-W  --route-table-name RT-2-LB-W --name TO-SPK2-W --output table

AddressPrefix    Name       NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ---------  ------------------  ----------------  -------------------  ---------------
10.2.0.0/24      TO-SPK2-W  10.0.0.254          VirtualAppliance  Succeeded            RG-FW-LAB-W
</pre>

### Show spoke VM effective route tables
<pre lang= >
<b>az network nic show-effective-route-table -g RG-FW-LAB-E --name VM1-SPK1-E-eth0 --output table</b>

Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.11.0.0/16      VnetLocal
Default   Active   10.10.0.0/16      VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
User      Active   10.11.0.0/24      VirtualAppliance  10.10.0.254 <b>>>>> LB VIP</b>
User      Active   10.12.0.0/24      VirtualAppliance  10.10.0.254 <b>>>>> LB VIP</b>


<b>az network nic show-effective-route-table -g RG-FW-LAB-E --name VM1-SPK2-E-eth0 --output table</b>

Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.12.0.0/16      VnetLocal
Default   Active   10.10.0.0/16      VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
User      Active   10.11.0.0/24      VirtualAppliance  10.10.0.254 <b>>>>> LB VIP</b>
User      Active   10.12.0.0/24      VirtualAppliance  10.10.0.254 <b>>>>> LB VIP</b>

<b>az network nic show-effective-route-table -g RG-FW-LAB-W --name VM1-SPK1-W-eth0 --output table</b>

Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.1.0.0/16       VnetLocal
Default   Active   10.0.0.0/16       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
User      Active   10.1.0.0/24       VirtualAppliance  10.0.0.254 <b>>>>> LB VIP</b>
User      Active   10.2.0.0/24       VirtualAppliance  10.0.0.254 <b>>>>> LB VIP</b>

<b>az network nic show-effective-route-table -g RG-FW-LAB-W --name VM1-SPK2-W-eth0 --output table</b>

Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.2.0.0/16       VnetLocal
Default   Active   10.0.0.0/16       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
User      Active   10.1.0.0/24       VirtualAppliance  10.0.0.254 <b>>>>> LB VIP</b>
User      Active   10.2.0.0/24       VirtualAppliance  10.0.0.254 <b>>>>> LB VIP</b>
</pre>

### Now that we checked the effective route, and they are showeing that to reach each spoke VNET, the next-hop is the LB VIP, we can test ICMP between spokes. We can also look at the firewall session table to view the flows.
<pre lang= >
<b>Ifconfig shows the VMs Ip address 10.1.0.4</b>
lab-user@W-SPK1-VM:~$ <b>ifconfig eth0</b>
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet <b>10.1.0.4</b>  netmask 255.255.255.0  broadcast 10.1.0.255
        inet6 fe80::20d:3aff:fe5b:1de0  prefixlen 64  scopeid 0x20<link>
        ether 00:0d:3a:5b:1d:e0  txqueuelen 1000  (Ethernet)
        RX packets 110477  bytes 111205356 (111.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 67573  bytes 13433299 (13.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

<b>Ping test from spoke 1 VM (10.1.0.4) to spoke 2 VM (10.2.0.5)</b>
lab-user@W-SPK1-VM:~$ <b>ping 10.2.0.4</b>
PING 10.2.0.4 (10.2.0.4) 56(84) bytes of data.
64 bytes from 10.2.0.4: icmp_seq=1 ttl=63 time=1.94 ms
64 bytes from 10.2.0.4: icmp_seq=2 ttl=63 time=1.58 ms
64 bytes from 10.2.0.4: icmp_seq=3 ttl=63 time=1.61 ms
</pre>

### This is what the firewall session table looks like for the ping above: Traffic allowed by policy: SPK1-TO-SPOK2/7 - Command output shows both wings of the session/connection.
<pre lang= >
lab-user@VSRX2-W# <b>run show security flow session</b>
Session ID: 30175, <b>Policy name</b>: <b>SPK1-TO-SPOK2/7</b>, Timeout: 2, Valid
  In: <b>10.1.0.4/9</b> --> <b>10.2.0.4/27945</b>;icmp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 1, Bytes: 84, <b><<< Inbound ICMP</b>
  Out: <b>10.2.0.4/27945</b> --> <b>10.1.0.4/9</b>;icmp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 1, Bytes: 84, <b>>>> Return ICMP</b>

Session ID: 30176, <b>Policy name</b>: <b>SPK1-TO-SPOK2/7</b>, Timeout: 2, Valid 
  In: <b>10.1.0.4/10</b> --> <b>10.2.0.4/27945</b>;icmp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 1, Bytes: 84, <b><<< Inbound ICMP</b>
  Out: <b>10.2.0.4/27945</b> --> <b>10.1.0.4/10</b>;icmp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 1, Bytes: 84, <b> >>> Return ICMP</b>
</pre>

