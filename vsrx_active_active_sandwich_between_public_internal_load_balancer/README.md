
### In this lab, we will sandwich two vSRX firewalls between a public and an internal Azure load balancer. This lab builds on my previous work which can be found <a href="https://github.com/ManCalAzure/AzureLabs/tree/master/vsrx_2_nva_active_active_with_public_load_balancer/README.md">here</a>.  Please complete the lab referenced before moving on to this one. <br /></p>

**In this lab we add an Azure internal load balancer on the TRUST side of the firewalls**

<pre lang= >
<b>1-</b> Create the internal load balancer, frontend-ip, and backend pool
<b>2-</b> Create the probe
<b>3-</b> Create the LB rule
<b>4-</b> Add the TRUST side vNICs to the backend pool
<b>5-</b> Split the vSRX routing into two instances
  - VR-TRUST
  - VR-UNTRUST
<b>6-</b> Configure each routing instance
<b>7-</b> Configure routing to support TRUST and UNTRUST LB Probes
<b>8-</b> Configure route leaking between TRUST and UNTRUST VRs to support transit
</pre>
<pre lang= >
<b>Create ILB with front end IP, and backend pool name</b>
az network lb create --resource-group RG-PLB-TEST --name ILB-1 --frontend-ip-name ILB-1-FE --private-ip-address 10.0.1.254 --vnet-name HUB-VNET --subnet O-TRUST --backend-pool-name ILB-BEPOOL --sku Standard

<b>Create the probe</b>
az network LB probe create --resource-group RG-PLB-TEST --name ILB-PROBE1 --protocol tcp --port 22 --interval 30 --threshold 2 --lb-name ILB-1

<b>Create the loab balancing rule</b>
az network lb rule create --resource-group RG-PLB-TEST --name ILB-R1-HAPORTS --backend-pool-name ILB-BEPOOL --probe-name ILB-PROBE1 --protocol all --frontend-port 0 --backend-port 0 --lb-name ILB-1
<b>Add vNICs to backend pool</b>
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX1-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet O-TRUST --lb-name ILB-1
az network nic ip-config update --resource-group RG-PLB-TEST --nic-name VSRX2-ge1 --name ipconfig1 --lb-address-pool ILB-BEPOOL --vnet-name HUB-VNET --subnet O-TRUST --lb-name ILB-1
</pre>
<pre lang= >
<b>We need to create an TRUST side NSG for traffic to flow. *Remember, utilizing Standard SKUs an NSG is required</b>
<b>Trust Subnet NSG</b>
az network nsg create --resource-group RG-PLB-TEST --name TRUST-NSG --location eastus
az network nsg rule create -g RG-PLB-TEST --nsg-name TRUST-NSG -n ALLOW-ALL --priority 200 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges '*' --access Allow --protocol '*' --description "Allow All to Trust Subnet"
<b>Associate Trust vNICs with TRUST-NSG</b>
az network nic update --resource-group RG-PLB-TEST --name VSRX1-ge1 --network-security-group TRUST-NSG
az network nic update --resource-group RG-PLB-TEST --name VSRX2-ge1 --network-security-group TRUST-NSG
</pre>
Te be continued..... work in progress....
