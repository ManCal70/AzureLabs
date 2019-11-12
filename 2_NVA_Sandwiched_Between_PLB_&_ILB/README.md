
### This lab builds on my previous work which can be found <a href="https://github.com/ManCalAzure/AzureLabs/blob/master/JunOS-To-Azure-VNG-IPSec+BGP/README.md">here</a>.  Please complete the lab referenced before moving on to this one. <br /></p>

**In this lab we add an Azure internal load balancer on the TRUST side of the firewalls**

<pre lang= >
<b>1-</b> Create the internal load balancer, frontend-ip, and backend pool
<b>2-</b> Create the probe
<b>3-</b> Create the LB rule
<b>4-</b> Add the TRUST side vNICs to the backend pool
<b>5-</b> Split the vSRX routing into two instances
  - VR-TRUST
  - VR-UNTRUST
<b>6-</b> Configure each routing instance
<b>7-</b> Configure routing to support TRUST and UNTRUST LB Probes
<b>8-</b> Configure route leaking between TRUST and UNTRUST VRs to support transit
</pre>


