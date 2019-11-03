# Azure Lab - Dual Juniper vSRX Active/Active + Azure Public LB

This lab will illustrate how to create an Azure Public load balancer, distribute traffic between two Juniper vSRX firewalls. The Azure public load balancer can be configured in two ways. 1) Default config - LB will translate the destination IP address to that of the BE pools VM (in this case vSRX), or 2)HA Ports config - This setting will NOT translate the incoming packets destination IP. This means the packets preserve its 5 tuples when being load balanced between the back end firewalls. I will show you how to configure both flavors of LB config.

### Design implications:
- When using the Azure Public LB default configuration, if you have multiple applications that are using the same destination port, you have to perform port translation. This is due to the fact that backed pool VMs are limited to one IP address. A NAT policy will need to be configured to perform the port translation. This can become cumbersome as you add more applications and create port translations. 

- Health check probes - There are 3 types of health probes you can use to check the health of the backend pool - TCP/HTTP/HTTPS - in this design, we are going to probe ssh to test firewalls responses. I will be enabling ssh service on the untrust interface where the probe will ingress. You should always secure the control plane by creating ACLs/Filters to only allow the required sources (that is beyond the scope of this document). Always keep in mind that probes are sent to the IP address of the firewalls, this means that when you are using the 'default' load balancer configuration, you may have conflics with firewall 'management' configurations (ssh etc...). 

- Floating IP configuration - Floating IP configuration will NOT perform destination NAT on the packets processed by the load balancer. The traffic will be load balanced and routed to the backend firewalls preserving the original 5 tuples. The firewall still requires a NAT rule which translates the destination IP address (Public load balancer IP address) to the private side resource. However, this configuration overcomes the multiple applications and port numbers limitations from the 'default' config (applications utilizing the same destination port). This style of configuration also mitigates the management traffic conflicts. 

# Default Config Topology Details
![alt text](https://github.com/ManCalAzure/AzureLabs/blob/master/Juniper_vSRX_HA_with_Public_LB/default-topo.png)



  
