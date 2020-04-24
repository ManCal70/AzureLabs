### Automation script to import all Office365 IP addresses into a UDR with NextHop 'Internet'

### Why did I publish this script? 
One use case which has become more common of late due to Covid-19, is in a Windows Virtual Desktop enviroment where customers are wanting to force tunnel traffic (advertise default from on-prem) from Azure to their on-prem. For security reasons, some customers want to force tunnel. This introduces some challenges with the desktops and also Office365.

Windows Virtual Desktop service requires that all desktops are able to connect via TCP 443 to the 'control plane'

<pre lang= >
<b>What the script does in order:</b>
1- Downloads all Office 365 IP addresses and stores them in a variable ($flatIp4s)
2- Checks the specific VNET in question for existing subnets
3- Create the route table names for each subnet with the subnet name + '-RT'
4- Creates a new route table if one does not already exit
5- Creates the routing configuration. Route table names and prefixes based on variable $flatIp4s
6- Add the Microsoft Windows Activation (KMS) service route as well
7- Applies route table to the subnet
8- Updates the VNET with the new subnet and configuration
</pre>
