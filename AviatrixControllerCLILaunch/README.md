Step by step process to launch an Aviatrix controller using Azure CLI

1.	Create a resource group
az group create --name RG-AVX-CONTROLLER --location eastus --output table
2.	Create a storage account for the Aviatrix controller bootdiags
az storage account create -n avxbootdiag -g RG-AVX-CONTROLLER -l eastus --sku Standard_LRS
3.	Create A VNET and Subnet

az network vnet create --name VNET-AVX-CONTROLLER --resource-group RG-AVX-CONTROLLER --location eastus --address-prefix 10.99.0.0/24

az network vnet subnet create --vnet-name VNET-AVX-CONTROLLER --name SUB1 --resource-group RG-AVX-CONTROLLER --address-prefixes 10.99.0.0/24 --output table

4.	Create a public IP object for the Aviatrix controller

az network public-ip create --name AVX-CONTROLLER --allocation-method Static --resource-group  RG-AVX-CONTROLLER --location eastus --sku Basic

5.	Create a vNIC for the controller and bind the public IP object create above

az network nic create --resource-group RG-AVX-CONTROLLER --location eastus --name AVX-CONTROLLER-eth0 --vnet-name VNET-AVX-CONTROLLER --subnet SUB1 --public-ip-address  AVX-CONTROLLER --private-ip-address 10.99.0.4

6.	Get the Aviatrix marketplace image list

az vm image list --all --publisher Aviatrix --output table

7.	Once you know the image you are going to use, accept the terms

az vm image terms accept --urn aviatrix-systems:aviatrix-bundle-payg:aviatrix-enterprise-bundle-byol:5.13.6

8.	Deploy the controller VM

az vm create --resource-group RG-AVX-CONTROLLER --location eastus --name AVX-CONTROLLER --size Standard_DS3_v2 --nics AVX-CONTROLLER-eth0 --image aviatrix-systems:aviatrix-bundle-payg:aviatrix-enterprise-bundle-byol:5.13.6 --admin-username lab-user --admin-password Ahina88f!!!! --boot-diagnostics-storage avxbootdiag --no-wait



This is the output you will see in the Azure portal:

 

Portal access:
 

