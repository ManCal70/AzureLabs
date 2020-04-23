#### This lab addresses Azure Windows Virtual Desktop and force tunneling of traffic challenges with Office365:
#### There are different ways to address some of these challenges, this is one example.

<pre lang= >
<b>In a WVD enviroment, there are two components you should take into account:</b>
1- The control plane - Web access, Gateway, Broker, LB, Management, Diagonostics-
2- The data plane - VNET traffic where desktops are deployed
</pre>
### Typical Azure WVD Enviroment
<table><tr><td>
    <img src="https://github.com/ManCalAzure/AzureLabs/blob/master/O365_IP_ADDRESSES_TO_UDR/wvd1.png" lt="" title="Lab Topology" width="850" height="500"  />
</td></tr></table>

#### In a WVD enviroment, where <b>forced tunneling</b> is required, this introduces two challenges:
##### 1- <b>Control Plane</b> destined traffic will take the default route via the on-prem connection, which is not efficient, or outright slow.
##### 2- <b>Office365</b> destined traffic will also take the default route via the on-prem connection. This can also introduce unwanted latency.

<b>*As shows Below:</b>
<table><tr><td>
    <img src="https://github.com/ManCalAzure/AzureLabs/blob/master/O365_IP_ADDRESSES_TO_UDR/wvd2.png" lt="" title="Lab Topology" width="850" height="500"  />
</td></tr></table>

