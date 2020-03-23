#### Azure Network Security Lab #2 - In this lab, we will deploy 2 NVA firewalls (Juniper vSRXs) with both a Public and Internal load balancer. This provides HA for both inbound and outbound connections. 

#### Topology
![](https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_HA_%2B_Az_Pub_%2B_Int_LB/topo-diagram.png)

### Lab Configuration Elements
<pre lang= >
<b>1-</b> Create a resource group
<b>2-</b> Create a storage account (bootdiags)
<b>3-</b> Create VNETS (hub and spokes)
<b>4-</b> Create Subnets
<b>5-</b> Create the VNET peerings between the hub and spokes
<b>6-</b> Create public IPs for the firewalls
<b>7-</b> Create vNICs (For firewalls & VMs)
<b>8-</b> Create control plane (management) and data plane Network Security Groups (UNTRUST & TRUST) (NSGs)
<b>9-</b> Associate the vNICs with their correponding NSGs
<b>10-</b> Create the firewall and Test web server
<b>11-</b> Create the Azure public load balancer
  - Backend poool
  - Probe
  - LB rule - with floating IP
  - Associate the firewall UNTRUST vNICs with the LB backendpool
<b>12-</b> Create the Azure internal load balancer
  - Backend poool
  - Probe
  - LB rule - with HA ports
  - Associate the firewall TRUST vNICs with the internal LB backendpool
<b>13-</b> Create the spoke UDR + Route which will route traffic to the internal LB VIP
<b>14-</b> Associate the UDR with the spoke subnet
<b>15-</b> Configure the firewalls and test web servers
</pre>

### Create a resource group
<pre lang= >
az group create --name RG-LB-TEST --location eastus --output table
</pre>

### Create a storage account for bootdiags
<pre lang= >
az storage account create -n mcbootdiag -g RG-LB-TEST -l eastus --sku Standard_LRS
</pre>

### Create the HUB and a SPOKE VNET
<pre lang= >
az network vnet create --name HUB-VNET --resource-group RG-LB-TEST --location eastus --address-prefix 10.0.0.0/16
az network vnet create --name SPOKE-VNET --resource-group RG-LB-TEST --location eastus --address-prefix 10.80.0.0/16
</pre>

### Create the Subnets in HUB and SPOKE VNETs
<pre lang= >
az network vnet subnet create --vnet-name HUB-VNET --name MGMT --resource-group RG-LB-TEST --address-prefixes 10.0.254.0/24 --output table
az network vnet subnet create --vnet-name HUB-VNET --name UNTRUST --resource-group RG-LB-TEST --address-prefixes 10.0.0.0/24 --output table
az network vnet subnet create --vnet-name HUB-VNET --name TRUST --resource-group RG-LB-TEST --address-prefixes 10.0.1.0/24 --output table
az network vnet subnet create --vnet-name SPOKE-VNET --name VMWORKLOADS --resource-group RG-LB-TEST --address-prefixes 10.80.99.0/24 --output table
</pre>

### VNET Peer HUB and SPOKE VNETs
<pre lang= >
az network vnet peering create -g RG-LB-TEST --name HUB-TO-SPOKE --vnet-name HUB-VNET --remote-vnet SPOKE-VNET --allow-forwarded-traffic --allow-vnet-access --output table
az network vnet peering create -g RG-LB-TEST --name SPOKE-TO-HUB --vnet-name SPOKE-VNET --remote-vnet HUB-VNET --allow-forwarded-traffic --allow-vnet-access --output table
</pre>

### Create the Public IPs - When utilizing Public IPs with Standard SKU, an NSG is required on the Subnet/vNIC. Two public IPs will be created per Firewall NVA, and 1 for the Public LB. 1) fxp0 - management interface 2) ge0 - UNTRUST/Interface facing interface
<pre lang= >
vSRX1
az network public-ip create --name VSRX1-PIP-1 --allocation-method Static --resource-group RG-LB-TEST --location eastus --sku Standard
az network public-ip create --name VSRX1-PIP-2 --allocation-method Static --resource-group RG-LB-TEST --location eastus --sku Standard
vSRX2
az network public-ip create --name VSRX2-PIP-1 --allocation-method Static --resource-group RG-LB-TEST --location eastus --sku Standard
az network public-ip create --name VSRX2-PIP-2 --allocation-method Static --resource-group RG-LB-TEST --location eastus --sku Standard
Az Load Balancer Public IP
az network public-ip create --name AZ-PUB-LB-PIP --allocation-method Static --resource-group RG-LB-TEST --location eastus --sku Standard
</pre>

### Create the vNICs
fxp0 = Out of band management interface on vSRXs
<pre lang= >
VSRX1
az network nic create --resource-group RG-LB-TEST --location eastus --name VSRX1-fxp0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX1-PIP-1 --private-ip-address 10.0.254.4 
az network nic create --resource-group RG-LB-TEST --location eastus --name VSRX1-ge0 --vnet-name HUB-VNET --subnet UNTRUST --public-ip-address  VSRX1-PIP-2 --private-ip-address 10.0.0.4 --ip-forwarding
az network nic create --resource-group RG-LB-TEST --location eastus --name VSRX1-ge1 --vnet-name HUB-VNET --subnet TRUST --private-ip-address 10.0.1.4 --ip-forwarding
VSRX2
az network nic create --resource-group RG-LB-TEST --location eastus --name VSRX2-fxp0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX2-PIP-1 --private-ip-address 10.0.254.5
az network nic create --resource-group RG-LB-TEST --location eastus --name VSRX2-ge0 --vnet-name HUB-VNET --subnet UNTRUST --public-ip-address  VSRX2-PIP-2 --private-ip-address 10.0.0.5
az network nic create --resource-group RG-LB-TEST --location eastus --name VSRX2-ge1 --vnet-name HUB-VNET --subnet TRUST --private-ip-address 10.0.1.5
Web Server VM
az network nic create --resource-group RG-LB-TEST --location eastus --name WEB-eth0 --vnet-name SPOKE-VNET --subnet VMWORKLOADS --private-ip-address 10.80.99.10
</pre>

### Create NSGs - Since I selected to use 'Standard' SKU public IP addresses explicitly defined NSG is required
<pre lang= >
Contral Plane NSG
az network nsg create --resource-group RG-LB-TEST --name CP-NSG --location eastus
az network nsg rule create -g RG-LB-FW-LAB --nsg-name CP-NSG -n ALLOW-SSH --priority 300 --source-address-prefixes Internet --destination-address-prefixes 10.0254.0/24 --destination-port-ranges 22 --access Allow --protocol Tcp --description "Allow SSH to Management Subnet"
az network nsg rule create -g RG-LB-FW-LAB --nsg-name CP-NSG -n ALLOW-ICMP --priority 301 --source-address-prefixes Internet --destination-address-prefixes 10.0.54.0/24 --destination-port-ranges * --protocol Icmp --description "Allow ICMP to FW OOB interface"
Untrust Subnet NSG
az network nsg create --resource-group RG-LB-TEST --name UNTRUST-NSG --location eastus
az network nsg rule create -g RG-LB-FW-LAB --nsg-name UNTRUST-NSG -n ALLOW-HTTP --priority 200 --source-address-prefixes * --source-port-ranges * --destination-address-prefixes * --destination-port-ranges 80 --access Allow --protocol Tcp --description "Allow HTTP to Untrust Subnet"
Associate vNICs with corresponding NSGs
az network nic update --resource-group RG-LB-TEST --name VSRX1-fxp0 --network-security-group CP-NSG
az network nic update --resource-group RG-LB-TEST --name VSRX2-fxp0 --network-security-group CP-NSG
az network nic update --resource-group RG-LB-TEST --name VSRX1-ge0 --network-security-group UNTRUST-NSG
az network nic update --resource-group RG-LB-TEST --name VSRX2-ge0 --network-security-group UNTRUST-NSG
</pre>

<b>Create ILB with front end IP, and backend pool name</b>
<pre lang= >
az network lb create --resource-group RG-PLB-TEST --name ILB-1 --frontend-ip-name ILB-1-FE --private-ip-address 10.0.1.254 --vnet-name HUB-VNET --subnet TRUST --backend-pool-name ILB-BEPOOL --sku Standard
</pre>
<b>Output after created:</b>
<pre lang= >
az network lb list -g RG-PLB-TEST --output table
Location    Name       ProvisioningState    ResourceGroup    ResourceGuid
----------  ---------  -------------------  ---------------  ------------------------------------
eastus      AZ-PUB-LB  Succeeded            RG-PLB-TEST      75055a40-5f78-4502-acf3-71a5e6ad952f
</pre>
<b>Create the probe</b>
<pre lang= >
az network LB probe create --resource-group RG-PLB-TEST --name ILB-PROBE1 --protocol tcp --port 22 --interval 30 --threshold 2 --lb-name ILB-1
</pre>

<b>Show the probe after created:</b>
<pre lang= >
az network lb probe list --resource-group RG-PLB-TEST --lb-name AZ-PUB-LB --output table
IntervalInSeconds    Name       NumberOfProbes    Port    Protocol    ProvisioningState    ResourceGroup
-------------------  ---------  ----------------  ------  ----------  -------------------  ---------------
30                   BE-PROBE1  2                 22      Tcp         Succeeded            RG-PLB-TEST
</pre>

<b>Create the loab balancing rule with 'HA Ports'</b>
az network lb rule create --resource-group RG-PLB-TEST --name ILB-R1-HAPORTS --backend-pool-name ILB-BEPOOL --probe-name ILB-PROBE1 --protocol all --frontend-port 0 --backend-port 0 --lb-name ILB-1
</pre>

<b>Show the rule created:</b>
<pre lang= >
az network lb rule list --lb-name AZ-PUB-LB -g RG-PLB-TEST --output table
BackendPort    DisableOutboundSnat    EnableFloatingIp    EnableTcpReset    FrontendPort    IdleTimeoutInMinutes    LoadDistribution    Name       Protocol    ProvisioningState    ResourceGroup
-------------  ---------------------  ------------------  ----------------  --------------  ----------------------  ------------------  ---------  ----------  -------------------  ---------------
80             False                  True                False             80              4                       Default             LB-RULE-1  Tcp         Succeeded            RG-PLB-TEST
<b>Add trust side vNICs to backend pool utilized by the ILB</b>
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX1-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet TRUST --lb-name ILB-1
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX2-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet TRUST --lb-name ILB-1
</pre>
<pre lang= >
<b>We need to create a TRUST side NSG for traffic to flow. *Always keep in mind, when utilizing Standard SKUs, an NSG is required</b>
<b>Trust Subnet NSG</b>
az network nsg create --resource-group RG-PLB-TEST --name TRUST-NSG --location eastus

<b>Trust Subnet NSG check</b>
az network nsg show -g RG-PLB-TEST --name TRUST-NSG --output table
Location    Name       ProvisioningState    ResourceGroup    ResourceGuid
----------  ---------  -------------------  ---------------  ------------------------------------
eastus      TRUST-NSG  Succeeded            RG-PLB-TEST      fcd7c257-be8e-497e-abc3-2575b190ed6c

<b>Create required NSG rule</b>
az network nsg rule create -g RG-PLB-TEST --nsg-name TRUST-NSG -n ALLOW-ALL --priority 200 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges '*' --access Allow --protocol '*' --description "Allow All to Trust Subnet"

<b>NSG Rule check</b>
az network nsg rule show --name ALLOW-ALL --nsg-name TRUST-NSG -g RG-PLB-TEST --output table
Name       ResourceGroup    Priority    SourcePortRanges    SourceAddressPrefixes    SourceASG    Access    Protocol    Direction    DestinationPortRanges    DestinationAddressPrefixes    DestinationASG
---------  ---------------  ----------  ------------------  -----------------------  -----------  --------  ----------  -----------  -----------------------  ----------------------------  ----------------
ALLOW-ALL  RG-PLB-TEST      200         *                   *                        None         Allow     *           Inbound      *                        *                             None


<b>Associate Trust vNICs with TRUST-NSG</b>
az network nic update --resource-group RG-PLB-TEST --name VSRX1-ge1 --network-security-group TRUST-NSG
az network nic update --resource-group RG-PLB-TEST --name VSRX2-ge1 --network-security-group TRUST-NSG
</pre>
<pre lang= >
<b>Now we need to create a user defined route (UDR) which routes traffic to the internal load balancer VIP address. This address is applied to any VNET where you want traffic to be routed via the ILB.</b>
az network route-table create  --name UDR-TO-ILB-1 --resource-group RG-PLB-TEST -l eastus

<b>UDR creationg check</b>
az network route-table show --name UDR-TO-ILB-1 -g RG-PLB-TEST --output table
DisableBgpRoutePropagation    Location    Name          ProvisioningState    ResourceGroup
----------------------------  ----------  ------------  -------------------  ---------------
False                         eastus      UDR-TO-ILB-1  Succeeded            RG-PLB-TEST

<b>UDR creation</b>
az network route-table route create --name DEFAULT-RT-TO-ILB -g RG-LB-TEST --route-table-name UDR-TO-ILB-1 --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.254

<b>Route creation check</b>
az network route-table route show -g RG-LB-TEST --name DEFAULT-RT-TO-ILB --route-table-name UDR-TO-ILB-1 --output table
AddressPrefix    Name               NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  -----------------  ------------------  ----------------  -------------------  ---------------
0.0.0.0/0        DEFAULT-RT-TO-ILB  10.0.1.254          VirtualAppliance  Succeeded            RG-PLB-TEST

<b>Once the UDR is created, associate it or apply it to the VMWORKLOADS subnet.</b>
az network vnet subnet update --vnet-name SPOKE-VNET --name VMWORKLOADS --resource-group RG-PLB-TEST --route-table UDR-TO-ILB-1

<b>After the UDR is applied to the subnet, you can check the effective route table to ensure the route is in effect. *Keep in mind after applying a UDR this can cake up to a minute to propagate.</b>
az network nic show-effective-route-table --name WEB-eth0 --resource-group RG-PLB-TEST --output table

At this point you can check the effective route table of the VM vNIC to ensure the default points to the ILB IP
az network nic show-effective-route-table -g RG-PLB-TEST -n WEB-eth0 --output table
</pre>

<b>Output of the Web server effective route table</b>
<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_HA_%2B_Az_Pub_%2B_Int_LB/Route-table.png)</kbd>

<pre lang= >
<b>You can do the same with Azure CLI</b>

az network nic show-effective-route-table -g RG-PLB-TEST -n WEB-eth0 --output table
Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.80.0.0/16      VnetLocal
Default   Active   10.0.0.0/16       VNetPeering
Default   Invalid  0.0.0.0/0         Internet
User      Active   0.0.0.0/0         VirtualAppliance  10.0.1.254

</pre>
#### At this point we have:
<p>
<b>1-</b>The ILB configured with front end IP, probes, LB rules, and we have added the vSRX vNICs to the BE pool<br />
<b>2-</b>The UDR with 0/0 (default) route is applied to the client/source/TRUST subnet pointing to the VIP<br />
<b>3-</b>We have also created a TRUST side NSG<br />
</p>

<b>*</b>Since the vSRX firewall now has LB's on both the TRUST and UNTRUST zones, each LB will send probes to health check the firewall. These probes will be originating/ingress via TRUST (Internal LB) and UNTRUST (Public LB) interfaces. We need to make some routing changes in the vSRX to handle probes coming from the same source IP, but that need to be routed back via its corresponding interface. In Junos, this is handled via the use of a 'Virtual Router' or L3 routing tables. In the previous lab, since we only had a Public LB, we were ok with a single virtual router (VR) since the probes were originating from a single side of the firewalls (UNTRUST). The addition of the Internal LB, creates the scenario where the probes will be originating on both sides of the firewall (TRUST and UNTRUST). We need to ensure these probes are routed back out the interface or zone where they originated. 

<pre lang= >
These are the updates required to the vSRX

</pre>
