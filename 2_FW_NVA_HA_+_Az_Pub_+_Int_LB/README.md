#### The previous lab (Internet facing firewall NVAs (Juniper vSRX Firewalls) in HA design) can be found <a href="https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_HA_%2B_Az_Pub_LB/README.md">here</a>.  Please complete the lab referenced before moving forward with this one. <br /></p>
<pre lang= >
<b>In this lab, we will:</b>
<b>1-</b> Create an Azure internal load balancer with:
  - Frontend-ip
  - Backend pool
  - Probe on port 22
  - Load balancer rule with 'HA Ports' >> HA ports allows for all IPs & ports to be forwarded to the vSRX firewalls
<b>2-</b> Add TRUST side firewall vNICs to the backend pool 
<b>3-</b> on vSRX configuration, add a second routing instance (VR/VRF) to handle the health probes coming from TRUST and UNTRUST (will elaborate later)
  - TRUST VR
  - UNTRUST VR
<b>4-</b> Create a UDR which sets 0/0 next-hop ILB VIP
<b>5-</b> Bind the UDR to the VMs subnet
<b>6-</b> Configure each routing instance
<b>7-</b> Configure routing to support TRUST and UNTRUST LB Probes
<b>8-</b> Configure route leaking between TRUST and UNTRUST VRs to support transit
</pre>
<b>End goal of this lab - 2 FWs sandwich between two Azure load balancers for HA</b>
<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_HA_%2B_Az_Pub_%2B_Int_LB/firewall_sandwich.png)</kbd>
<pre lang= >
<b>Create ILB with front end IP, and backend pool name</b>
az network lb create --resource-group RG-PLB-TEST --name ILB-1 --frontend-ip-name ILB-1-FE --private-ip-address 10.0.1.254 --vnet-name HUB-VNET --subnet O-TRUST --backend-pool-name ILB-BEPOOL --sku Standard

<b>Output after created:</b>
az network lb list -g RG-PLB-TEST --output table
Location    Name       ProvisioningState    ResourceGroup    ResourceGuid
----------  ---------  -------------------  ---------------  ------------------------------------
eastus      AZ-PUB-LB  Succeeded            RG-PLB-TEST      75055a40-5f78-4502-acf3-71a5e6ad952f

<b>Create the probe</b>
az network LB probe create --resource-group RG-PLB-TEST --name ILB-PROBE1 --protocol tcp --port 22 --interval 30 --threshold 2 --lb-name ILB-1

<b>Show the probe after created:</b>
az network lb probe list --resource-group RG-PLB-TEST --lb-name AZ-PUB-LB --output table
IntervalInSeconds    Name       NumberOfProbes    Port    Protocol    ProvisioningState    ResourceGroup
-------------------  ---------  ----------------  ------  ----------  -------------------  ---------------
30                   BE-PROBE1  2                 22      Tcp         Succeeded            RG-PLB-TEST

<b>Create the loab balancing rule with 'HA Ports'</b>
az network lb rule create --resource-group RG-PLB-TEST --name ILB-R1-HAPORTS --backend-pool-name ILB-BEPOOL --probe-name ILB-PROBE1 --protocol all --frontend-port 0 --backend-port 0 --lb-name ILB-1

<b>Show the rule created:</b>
az network lb rule list --lb-name AZ-PUB-LB -g RG-PLB-TEST --output table
BackendPort    DisableOutboundSnat    EnableFloatingIp    EnableTcpReset    FrontendPort    IdleTimeoutInMinutes    LoadDistribution    Name       Protocol    ProvisioningState    ResourceGroup
-------------  ---------------------  ------------------  ----------------  --------------  ----------------------  ------------------  ---------  ----------  -------------------  ---------------
80             False                  True                False             80              4                       Default             LB-RULE-1  Tcp         Succeeded            RG-PLB-TEST
<b>Add trust side vNICs to backend pool utilized by the ILB</b>
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX1-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet O-TRUST --lb-name ILB-1
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX2-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet O-TRUST --lb-name ILB-1
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
az network route-table route create --name DEFAULT-RT-TO-ILB -g RG-PLB-TEST --route-table-name UDR-TO-ILB-1 --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.254

<b>Route creation check</b>
az network route-table route show -g RG-PLB-TEST --name DEFAULT-RT-TO-ILB --route-table-name UDR-TO-ILB-1 --output table
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
