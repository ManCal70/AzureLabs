#### JunOS SRX/vSRX IPSec configuration to interoperate with Azure Virtual Network Gateway (VNG)

#### This example includes BGP peering configuration.

<pre lang= >
<b>1-</b> Create a resource group
<b>2-</b> Create the VNET with GatewaySubnet
<b>3-</b> Create a public IP for the VNG
<b>4-</b> Create the VNG with the following parameters:
  - Sku = VpnGw1
  - Gateway-type = Vpn
  - vpn-type = RouteBased
  - ASN = 65002
  - bgp-peering-address = 10.225.254.254 (GatewaySubnet highest IP)
<b>5-</b> Create the 'Local Network Gateway' - remote firewall settings
  - gateway-ip-address = 71.59.10.124
  - asn = 65001 
  - bgp-peering-address = 10.250.250.250
  - local-address-prefixes = 10.250.0.0/16
<b>6-</b> Create the connection to tie the VNG and remote gateway in IPSec
  - vnet-gateway1 = GW-TEST-VNG
  - local-gateway2 = LGW-1
  - enable-bgp
</pre>
<pre lang= >
<b>Create Resource group</b>
az group create --name RG-GW-TEST --location westus

<b>Create VNET</b>
az network vnet create -n GW-TEST  -g RG-GW-TEST -l westus --address-prefix 10.225.0.0/16  --subnet-name GatewaySubnet --subnet-prefix 10.225.254.0/24

<b>Create VNG PIP</b>
az network public-ip create -n GW-TEST-PIP -g RG-GW-TEST --allocation-method Dynamic

<b>Create VPN GW - (for bgp peer address if you are setting this yourself grab the highest IP in the GatewaySubnet range .254)</b>
az network vnet-gateway create -n GW-TEST-VNG -l westus --public-ip-address GW-TEST-PIP -g RG-GW-TEST --vnet GW-TEST --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --asn 65002 --bgp-peering-address 10.225.254.254 --no-wait

<b>Create the Local Network Gateway (Remote firewall config): gw ip:71.59.10.124,remote asn 65001, peer ip 10.250.250.250, remote LAN 10.250.0.0/16</b>
az network local-gateway create --gateway-ip-address 71.59.10.124 -g RG-GW-TEST -n LGW-1 --asn 65001 --bgp-peering-address 10.250.250.250 --local-address-prefixes 10.250.0.0/16

<b>Create the Connection</b>
az network vpn-connection create -g RG-GW-TEST -n CONNECITON-1 --vnet-gateway1 GW-TEST-VNG --local-gateway2 LGW-1 --enable-bgp --location westus --shared-key AzLabPass123
</pre>
