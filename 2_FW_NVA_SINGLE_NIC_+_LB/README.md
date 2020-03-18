<p align="left">
<b>Design</center></b>
<p><p align="left"><b>2 single vNIC Network Virtual Appliances (NVA) Firewalls (Juniper vSRX) in Active/Active design front ended by an Internal LB. A single vNIC Firewall is useful to avoid having to use NAT. The way Azure LB hashing works, allows for flow affinity without the need of NAT. This use case only applies to traffic originating and destined to Azure VNETs. Internet bound traffic requires a different design.</b></p>

<p align="left"><b>This design can be applied across multiple 3rd party NVA vendors</p></b>
</p>
<b>In the following lab we will:</b>
<pre lang= >
* Create a resource group for all of the objects (LB, FW, VNET,...)
* Create a storagea account for boot diagnostics 
* Create a VNET w/ IP range
* Create 3 Subnets (Management, TRUST, and UNTRUST)
* Create public IP address objects for required elements
* Create nNICs for the firewalls (vSRX) and web server (Ubuntu + Apache)
* Create the virtual machines
* Configure the vSRX firewalls
* Create the Azure public load balancer
* Test Apache2 connectivity 
* Show the firewall session tables
</pre>

### Key details
