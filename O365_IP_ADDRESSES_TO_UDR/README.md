### Automation script to import all Office365 IP addresses into a UDR with NextHop 'Internet'
#### Here is what it does in order
1- Downloads all Office 365 IP addresses and stores them in a variable ($flatIp4s)
2- Checks the specific VNET in question for existing subnets
3- Create the route table names for each subnet with the subnet name + '-RT'
4- Creates a new route table if one does not already exit
5- Creates the routing configuration. Route table names and prefixes based on variable $flatIp4s
6- Add the Microsoft Windows Activation (KMS) service route as well
7- Applies route table to the subnet
8- Updates the VNET with the new subnet and configuration

