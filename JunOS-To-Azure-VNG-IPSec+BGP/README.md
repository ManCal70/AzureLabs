### JunOS SRX/vSRX IPSec configuration to interoperate with Azure Virtual Network Gateway (VNG). This example includes BGP peering configuration.

<b>This document assumes you already have a 'Resource Group', a VNET with a 'GatewaySubnet' configured. In the following lab we will:</b>
<pre lang= >
1- Create a VNG type VPN
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

**Screenshot of Azure portal example to create the Virtual Network Gateway (VNG)**

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/JunOS-To-Azure-VNG-IPSec%2BBGP/gw-view.png)</kbd>
<pre lang= >
<b>create VNET</b>
az network vnet create -n GW-TEST  -g RG-GW-TEST -l westus --address-prefix 10.225.0.0/16  --subnet-name GatewaySubnet --subnet-prefix 10.225.254.0/24

<b>Create VNG PIP</b>
az network public-ip create -n GW-TEST-PIP -g RG-GW-TEST --allocation-method Dynamic 

<b>Create VPN GW - (for bgp peer address if you are setting this yourself grab the highest IP in the GatewaySubnet range .254)</b>
az network vnet-gateway create -n GW-TEST-VNG -l westus --public-ip-address GW-TEST-PIP -g RG-GW-TEST --vnet GW-TEST --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --asn 65002 --bgp-peer-address 10.225.254.254 --no-wait
</pre>
