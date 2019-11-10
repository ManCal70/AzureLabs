### JunOS SRX/vSRX IPSec configuration to interoperate with Azure Virtual Network Gateway (VNG). This example includes BGP peering configuration.

<b>In the following lab we will:</b>
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

**Create the Virtual Network Gateway (VNG)**

<kbd>![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/JunOS-To-Azure-VNG-IPSec%2BBGP/vng-setup.png)</kbd>


