<b>Design</center></b>
<p><p align="left"><b>2 single vNIC Network Virtual Appliances (NVA) Firewalls (Juniper vSRX) in Active/Active design front ended by an Internal LB. A single vNIC Firewall is useful to avoid having to use NAT. The way Azure LB hashing works, allows for flow affinity without the need of NAT. This use case only applies to traffic originating and destined to Azure VNETs. Internet bound traffic requires a different design.</b></p>

<p align="center">
<b>Topology</center></b>

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/2_FW_NVA_SINGLE_NIC_+_LB/topology.png)</kbd>
<p align="left">


<p align="left"><b>This design can be applied across multiple 3rd party NVA vendors that are able to support intra-zone security policies and enforcement</p></b>
</p>
<b>The following elements need to be configured:</b>
<pre lang= >
* Create a resource group for all of the objects (LB, FW, VNET,...)
* Create a storagea account for boot diagnostics 
* Create 6 VNETs w/ IP ranges
* Create 6 Subnets (MGT, FWSUB, and vm subnsets)
* Create public IP address objects for required elements (Firewall management)
* Create nNICs for the firewalls (vSRX) and testing VMs (Ubuntu + Apache)
* Create the virtual machines (Firewalls and test VMs)
* Create the Network security groups, and apply to their respective subnets
* Create internal load balancer
* Create the UDR which is applied to the Vm subnets to route 0/0 traffic to the LB VIP
* Configure the vSRX firewalls
* Test connectivity between spokes 
* Show the firewall session tables
</pre>

### Key details