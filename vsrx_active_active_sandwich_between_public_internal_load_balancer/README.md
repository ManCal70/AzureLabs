The previous lab can be found <a href="https://github.com/ManCalAzure/AzureLabs/tree/master/vsrx_2_nva_active_active_with_public_load_balancer/README.md">here</a>.  Please complete the lab referenced before moving forward with this one. <br /></p>
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

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/vsrx_active_active_sandwich_between_public_internal_load_balancer/firewall_sandwich.png)</kbd>
<pre lang= >
<b>Create ILB with front end IP, and backend pool name</b>
az network lb create --resource-group RG-PLB-TEST --name ILB-1 --frontend-ip-name ILB-1-FE --private-ip-address 10.0.1.254 --vnet-name HUB-VNET --subnet O-TRUST --backend-pool-name ILB-BEPOOL --sku Standard
<b>Create the probe</b>
az network LB probe create --resource-group RG-PLB-TEST --name ILB-PROBE1 --protocol tcp --port 22 --interval 30 --threshold 2 --lb-name ILB-1

<b>Create the loab balancing rule with 'HA Ports'</b>
az network lb rule create --resource-group RG-PLB-TEST --name ILB-R1-HAPORTS --backend-pool-name ILB-BEPOOL --probe-name ILB-PROBE1 --protocol all --frontend-port 0 --backend-port 0 --lb-name ILB-1
<b>Add trust side vNICs to backend pool utilized by the ILB</b>
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX1-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet O-TRUST --lb-name ILB-1
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX2-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet O-TRUST --lb-name ILB-1
</pre>
<pre lang= >
<b>We need to create an TRUST side NSG for traffic to flow. *Always keep in mind, when utilizing Standard SKUs, an NSG is required</b>
<b>Trust Subnet NSG</b>
az network nsg create --resource-group RG-PLB-TEST --name TRUST-NSG --location eastus
az network nsg rule create -g RG-PLB-TEST --nsg-name TRUST-NSG -n ALLOW-ALL --priority 200 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges '*' --access Allow --protocol '*' --description "Allow All to Trust Subnet"
<b>Associate Trust vNICs with TRUST-NSG</b>
az network nic update --resource-group RG-PLB-TEST --name VSRX1-ge1 --network-security-group TRUST-NSG
az network nic update --resource-group RG-PLB-TEST --name VSRX2-ge1 --network-security-group TRUST-NSG
</pre>
<pre lang= >
<b>Now we need to create a user defined route (UDR) which routes traffic to the internal load balancer VIP address. This address is applied to any VNET where you want traffic to be routed via the ILB.</b>
az network route-table create  --name UDR-TO-ILB --resource-group RG-PLB-TEST -l eastus
az network route-table route create --name DEFAULT-TO-ILB -g RG-PLB-TEST --route-table-name UDR-TO-ILB --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.254
</pre>
Te be continued..... work in progress....
