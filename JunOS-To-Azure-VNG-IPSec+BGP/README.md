### JunOS SRX/vSRX IPSec configuration to interoperate with Azure Virtual Network Gateway (VNG). This example includes BGP peering configuration.

<pre lang= >
1- Create a VNG type VPN

</pre>

**Screenshot of Azure portal example to create the Virtual Network Gateway (VNG)**

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/JunOS-To-Azure-VNG-IPSec%2BBGP/gw-view.png)</kbd>
<b>**CLI examples**</b>
<pre lang= >
<b>Create Resource group</b>
az group create --name RG-GW-TEST --location wastus
<b>Create VNET</b>
az network vnet create -n GW-TEST  -g RG-GW-TEST -l westus --address-prefix 10.225.0.0/16  --subnet-name GatewaySubnet --subnet-prefix 10.225.254.0/24

<b>Create VNG PIP</b>
az network public-ip create -n GW-TEST-PIP -g RG-GW-TEST --allocation-method Dynamic 

<b>Create VPN GW - (for bgp peer address if you are setting this yourself grab the highest IP in the GatewaySubnet range .254)</b>
az network vnet-gateway create -n GW-TEST-VNG -l westus --public-ip-address GW-TEST-PIP -g RG-GW-TEST --vnet GW-TEST --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --asn 65002 --bgp-peer-address 10.225.254.254 --no-wait
</pre>
