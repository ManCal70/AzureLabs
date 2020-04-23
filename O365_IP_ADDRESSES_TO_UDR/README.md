#### This lab addresses Azure Windows Virtual Desktop and force tunneling of traffic challenges with Office365:
#### There are different ways to address some of these challenges, this is one example.

<pre lang= >
<b>In a WVD enviroment, there are two components you should take into account:</b>
1- The control plane - Web access, Gateway, Broker, LB, Management, Diagonostics-
2- The data plane - VNET traffic where desktops are deployed
</pre>
<table><tr><td>
    <img src="https://github.com/ManCalAzure/AzureLabs/blob/master/O365_IP_ADDRESSES_TO_UDR/single-vnic-topo.png" lt="" title="Lab Topology" width="850" height="500"  />
</td></tr></table>

