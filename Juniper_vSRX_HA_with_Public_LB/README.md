# Azure Lab - Dual Juniper vSRX Active/Active + Azure Public LB

This lab will illustrate how to create an Azure Public load balancer, distribute traffic between two Juniper vSRX firewalls. The Azure public load balancer can be configured in two ways. 1) Default config - LB will translate the destination IP address to that of the BE pools VM (in this case vSRX), or 2)HA Ports config - This setting will NOT translate the incoming packets destination IP. This means the packets preserve its 5 tuples when being load balanced between the back end firewalls. 

Design implications:
  ##Default config$$ - This will require 
