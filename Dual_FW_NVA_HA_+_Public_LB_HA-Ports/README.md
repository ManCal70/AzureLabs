# Azure Lab - Dual NVA Firewalls (Juniper vSRX) Active/Active + Azure Public LB & Floating IP 
### This design can be applied across multiple 3rd party NVA vendors. 

<b>In the following lab we will:</b>
<pre lang= >
1- Create a resource group for all of the objects (LB, FW, VNET,...)
2- Create a VNET w/ IP range
3- Create 3 Subnets (Management, TRUST, and UNTRUST)
4- Create public IP address objects for required elements
5- Create nNICs for the firewalls (vSRX) and web server (Ubuntu + Apache)
6- Create the virtual machines
7- Configure the vSRX firewalls
8- Create the Azure public load balancer
9- Test Apache2 connectivity 
10-Show the firewall session tables
</pre>

The Azure public load balancer can be configured in two ways. 1) Default config - LB will translate the destination IP address to that of the BE pools VM (in this case vSRX), or 2)HA Ports config - This setting will NOT translate the incoming packets destination IP. This means the packets preserve its 5 tuples when being load balanced between the back end firewalls. This post will focus on HA ports configuration. 

### Design implications:
- When using the Azure Public LB default configuration, if you have multiple applications that are using the same destination port, you have to perform port translation. This is due to the fact that backed pool VMs are limited to one IP address. A NAT policy will need to be configured to perform the port translation. This can become cumbersome as you add more applications and create port translations. 

- Health check probes - There are 3 types of health probes you can use to check the health of the backend pool - TCP/HTTP/HTTPS - in this design, we are going to probe ssh to test firewalls responses. I will be enabling ssh service on the untrust interface where the probe will ingress. You should always secure the control plane by creating ACLs/Filters to only allow the required sources (that is beyond the scope of this document). Always keep in mind that probes are sent to the IP address of the firewalls, this means that when you are using the 'default' load balancer configuration, you may have conflics with firewall 'management' configurations (ssh etc...). 

- Floating IP configuration - Floating IP configuration will NOT perform destination NAT on the packets processed by the load balancer. The traffic will be load balanced and routed to the backend firewalls preserving the original 5 tuples. The firewall still requires a NAT rule which translates the destination IP address (Public load balancer IP address) to the private side resource. However, this configuration overcomes the multiple applications and port numbers limitations from the 'default' config (applications utilizing the same destination port). This style of configuration also mitigates the management traffic conflicts. 

# Topology Details - Simple Trust and Untrust topology. This is also applicable if the VM was running on a peered VNET (Spoke)

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/Dual_FW_NVA_HA_%2B_Public_LB_HA-Ports/default-topology.png)</kbd>

**Elements required**
<pre lang= >
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
  - VNET
      - IP Range - 10.0.0.0/16
      - Management Subnet - MGMT 10.0.254.0/24
      - UNTRUST Subnet - O-UNTRUST 10.0.0.0/24
      - TRUST Subnet - O-TRUST 10.0.1.0/24
    - Ubuntu Virtual machine + Apache2
</pre>

**Create the Resource Group**
<pre lang= >
az group create --name RG-PLB-TEST --location eastus
</pre>

**Create the Hub VNET**
<pre lang= >
az network vnet create --name HUB-VNET --resource-group RG-PLB-TEST --location eastus --address-prefix 10.0.0.0/16
</pre>

**Create the Subnets**
<pre lang= >
az network vnet subnet create --vnet-name HUB-VNET --name MGMT --resource-group RG-PLB-TEST --address-prefixes 10.0.254.0/24
az network vnet subnet create --vnet-name HUB-VNET --name O-UNTRUST --resource-group RG-PLB-TEST --address-prefixes 10.0.0.0/24
az network vnet subnet create --vnet-name HUB-VNET --name O-TRUST --resource-group RG-PLB-TEST --address-prefixes 10.0.1.0/24
</pre>

**Create the Public IPs - When utilizing Public IPs with Standard SKU, an NSG is required on the Subnet/vNIC. Two public IPs will be created per Firewall NVA, and 1 for the Public LB. 1) fxp0 - management interface 2) ge0 - UNTRUST/Interface facing interface**
<pre lang= >
az network public-ip create --name VSRX1-PIP-1 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
az network public-ip create --name VSRX1-PIP-2 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard

az network public-ip create --name VSRX2-PIP-1 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
az network public-ip create --name VSRX2-PIP-2 --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
<b>Az Load Balancer Public IP</b>
az network public-ip create --name AZ-PUB-LB-PIP --allocation-method Static --resource-group RG-PLB-TEST --location eastus --sku Standard
</pre>
**Create the vNICs**
* fxp0 = Out of band management interface on vSRXs
<pre lang>
<b>VSRX1</b>
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX1-fxp0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX1-PIP-1 --private-ip-address 10.0.254.4
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX1-ge0 --vnet-name HUB-VNET --subnet O-UNTRUST --public-ip-address  VSRX1-PIP-2 --private-ip-address 10.0.0.4
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX1-ge1 --vnet-name HUB-VNET --subnet O-TRUST --private-ip-address 10.0.1.4
<b>VSRX2</b>
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX2-fxp0 --vnet-name HUB-VNET --subnet MGMT --public-ip-address  VSRX2-PIP-1 --private-ip-address 10.0.254.5
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX2-ge0 --vnet-name HUB-VNET --subnet O-UNTRUST --public-ip-address  VSRX2-PIP-2 --private-ip-address 10.0.0.5
az network nic create --resource-group RG-PLB-TEST --location eastus --name VSRX2-ge1 --vnet-name HUB-VNET --subnet O-TRUST --private-ip-address 10.0.1.5
<b>Web Server VM</b>
az network nic create --resource-group RG-PLB-TEST --location eastus --name WEB-eth0 --vnet-name HUB-VNET --subnet O-TRUST --private-ip-address 10.0.1.10
</pre>
**Create the vSRX firewall VM**
<pre lang=>
<b>First - Accept the Juniper Networks license agreement</b>
Get-AzureRmMarketplaceTerms -Publisher juniper-networks -Product vsrx-next-generation-firewall -Name vsrx-byol-azure-image | Set-AzureRmMarketplaceTerms -Accept
<b>VSRX1</b>
az vm create --resource-group RG-PLB-TEST --location eastus --name VSRX1 --size Standard_DS3_v2 --nics VSRX1-fxp0 VSRX1-ge0 VSRX1-ge1 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234
<b>VSRX2</b>
az vm create --resource-group RG-PLB-TEST --location eastus --name VSRX2 --size Standard_DS3_v2 --nics VSRX2-fxp0 VSRX2-ge0 VSRX2-ge1 --image juniper-networks:vsrx-next-generation-firewall:vsrx-byol-azure-image:19.2.1 --admin-username lab-user --admin-password AzLabPass1234
</pre>
**Create a test Web server VM**
<pre lang=>
az vm create -n WEB-SERVER -g RG-PLB-TEST --image UbuntuLTS --admin-username lab-user --admin-password AzLabPass1234 --nics WEB-eth0
</pre>
**Create the Azure Public load balancer**
<pre lang= >
<b>Create the LB</b>
az network lb create --resource-group RG-PLB-TEST --name AZ-PUB-LB --sku Standard --public-ip-address AZ-PUB-LB-PIP
<b>Create the backend pool</b>
az network LB address-pool create --lb-name AZ-PUB-LB --name PLB1-BEPOOL --resource-group RG-PLB-TEST
<b>Create the probe</b>
az network LB probe create --resource-group RG-PLB-TEST --name BE-PROBE1 --protocol tcp --port 22 --interval 30 --threshold 2 --lb-name AZ-PUB-LB
<b>Create a LB rule</b>
az network lb rule create --resource-group RG-PLB-TEST --name LB-RULE-1 --backend-pool-name PLB1-BEPOOL --probe-name BE-PROBE1 --protocol Tcp --frontend-port 80 --backend-port 80 --lb-name AZ-PUB-LB --floating-ip true --output table
<b>Add the VSRX1-ge0 & VSRX2-ge0 vNICs to the LB backend pool</b>
az network nic ip-config update -g RG-PLB-TEST --nic-name VSRX1-ge0 -n ipconfig1 --lb-address-pool PLB1-BEPOOL --vnet-name hub-vnet --subnet O-UNTRUST --lb-name AZ-PUB-LB
az network nic ip-config update -g RG-PLB-TEST --nic-name VSRX2-ge0 -n ipconfig1 --lb-address-pool PLB1-BEPOOL --vnet-name hub-vnet --subnet O-UNTRUST --lb-name AZ-PUB-LB
</pre>

**vSRX configuraitons- Both vSRX will have identical configs**
<pre lang= >
<b>Interfaces configuration</b>
set interfaces ge-0/0/0 description UNTRUST
set interfaces ge-0/0/0 unit 0 family inet dhcp
set interfaces ge-0/0/1 description TRUST
set interfaces ge-0/0/1 unit 0 family inet dhcp
set interfaces fxp0 unit 0
<b>Routing instance configuration</b>
set routing-instances VR-1 instance-type virtual-router
set routing-instances VR-1 routing-options static route 168.63.129.16/32 next-hop 10.0.0.1  >><b>LB probe static route</b>
set routing-instances VR-1 routing-options static route 0.0.0.0/0 next-hop 10.0.0.1 >><b>Default route to internet</b>
set routing-instances VR-1 interface ge-0/0/0.0
set routing-instances VR-1 interface ge-0/0/1.0
<b>Security zone configuraiton</b>
set security zones security-zone TRUST address-book address 10.0.1.10/32 10.0.1.10/32 >><b>Address book entry of web server</b>
set security zones security-zone TRUST interfaces ge-0/0/1.0 host-inbound-traffic system-services all
set security zones security-zone TRUST interfaces ge-0/0/1.0 host-inbound-traffic protocols all
set security zones security-zone UNTRUST interfaces ge-0/0/0.0 host-inbound-traffic system-services dhcp
set security zones security-zone UNTRUST interfaces ge-0/0/0.0 host-inbound-traffic system-services ssh
<b>Destination NAT (DNAT)</b>
set security nat destination pool DST-NAT-POOL-1 address 10.0.1.10/32 >><b>IP address of Web server</b>
set security nat destination rule-set DST-RS1 from interface ge-0/0/0.0 >><b>Ingress interface of traffic</b>
set security nat destination rule-set DST-RS1 rule DST-R1 match destination-address 52.xx.xx.xx/32 >><b>Public IP of LB</b>
<b>Source NAT (SNAT) for return flow affinity</b>
set security nat source rule-set SNAT-FOR-DNAT-TO-WORK from zone UNTRUST
set security nat source rule-set SNAT-FOR-DNAT-TO-WORK to zone TRUST
set security nat source rule-set SNAT-FOR-DNAT-TO-WORK rule SNAT-R1 match destination-address 10.0.1.0/24
set security nat source rule-set SNAT-FOR-DNAT-TO-WORK rule SNAT-R1 then source-nat interface
<b>Security policies to allow incoming HTTP traffic to the Web server</b>
set security policies from-zone UNTRUST to-zone TRUST policy DST-TO-WEB-TEST match source-address any
set security policies from-zone UNTRUST to-zone TRUST policy DST-TO-WEB-TEST match destination-address 10.0.1.10/32
set security policies from-zone UNTRUST to-zone TRUST policy DST-TO-WEB-TEST match application junos-http
set security policies from-zone UNTRUST to-zone TRUST policy DST-TO-WEB-TEST then permit
set security policies from-zone UNTRUST to-zone TRUST policy DST-TO-WEB-TEST then log session-init
set security policies from-zone UNTRUST to-zone TRUST policy DST-TO-WEB-TEST then log session-close
</pre>

**View of the vSRX session table**
<pre lang= >
* Health probe session shows the Azure probe source address destined to 10.0.0.4 (vSRX UNTRUST vNIC IP)
<b>show security flow session</b> 
Session ID: 111891, Policy name: self-traffic-policy/1, Timeout: 1798, Valid
<b>Incoming connection</b>In: <b>168.63.129.16/57166</b> --> 10.0.0.4/22;tcp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 3, Bytes: 132, 
<b>Outgoing connection</b>Out: 10.0.0.4/22 --> 168.63.129.16/57166;tcp, Conn Tag: 0x0, If: .local..7, Pkts: 2, Bytes: 112, 
Total sessions: 1

<b>This output shows the incoming HTTP connection to the LB Public IP</b>
*Since we have "Floating IP" enabled on the LB rule, the LB performs no destination translation

Session ID: 111929, Policy name: DST-TO-WEB-TEST/6, Timeout: 298, Valid
<b>Incoming connection</b>In: 71.59.10.124/19208 --> <b>52.xx.xx.xx</b>/80;tcp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 6, Bytes: 1055, 
<b>Outgoing connection</b>Out: 10.0.1.10/80 --> 10.0.1.4/28363;tcp, Conn Tag: 0x0, If: ge-0/0/1.0, Pkts: 8, Bytes: 7524, 
Total sessions: 2
</pre>
**Test connection to the backend Web server via the Public LB IP address - It works ;) you can shut down a vSRX and traffic will continue to flow**

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/Dual_FW_NVA_HA_%2B_Public_LB_HA-Ports/apache.png)</kbd>

