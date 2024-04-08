<h1 align="center">
  <br>
  Aks and Application Gateway Ingress Controller configration 
  
  <br>
</h1>

# Step by step
``` 
Create Resource group Vnet ad Sub-Net 

```

1. **Create a virtual network and subnet**

*** Application Gateway Variables ***

export pipName=intel-dev-pip
export appgwRgName=DevAgw-rg
export appgwVnetName=dev-appgw-vnet
export appgwSnetName=dev-appgw-snet-01
export location=centralindia
export appgwVnetPrefix=10.9.0.0/24
export appgwSnetPrefix=10.9.0.0/26
export wafPolicyName=dev-waf-policy

1.1. Resource group, Vnet and Subnet creation for Application Gateway

1.1.1. Create a resource group using the [az group create](https://learn.microsoft.com/en-us/cli/azure/group#az_group_create) command.
az group create --name $appgwRgName --location $location

1.1.2. If you don't have an existing virtual network and subnet to use, create these network resources using the [az network vnet create](https://learn.microsoft.com/en-us/cli/azure/network/vnet#az_network_vnet_create) command. The following example command creates a virtual network named myAKSVnet with the address prefix of 192.168.0.0/16 and a subnet named myAKSSubnet with the address prefix 192.168.1.0/24:

az network vnet create --resource-group $appgwRgName --name $appgwVnetName --address-prefixes $appgwVnetPrefix --subnet-name $appgwSnetName --subnet-prefix $appgwSnetName

1.1.3 Create WAF policy
az network application-gateway waf-policy create --name $wafPolicyName --resource-group $appgwRgName

1.1.4 Create public ip
az network public-ip create -n $pipName -g $appgwRgName -l $location --allocation-method Static --sku Standard

1.1.5. Create application gateway
az network application-gateway create -n $appgwRgName -l $location -g $appgwRgName --sku WAF_v2 --public-ip-address $pipName --vnet-name $appgwVnetName --subnet $appgwSnetName --priority 100 --waf-policy $wafPolicyName

1.2.1 Resource group, Vnet and Subnet creation for AKS

*** AKS Variables ***

export rgName=IntelDevAks-rg 
export location=centralindia
export vnetName=gg-centralindia-vnet
export vnetAddPrefix=10.10.0.0/16
export subnetName=gg-il-app01-subnet-01
export subnetPrefix=10.10.0.0/26
export aksName=gg-il-app01
export nodeVmSize=Standard_B2ms



2.1.1 Create a resource group using the [az group create](https://learn.microsoft.com/en-us/cli/azure/group#az_group_create) command.
az group create --name $rgName --location $location

2.2.2 If you don't have an existing virtual network and subnet to use, create these network resources using the [az network vnet create](https://learn.microsoft.com/en-us/cli/azure/network/vnet#az_network_vnet_create) command. The following example command creates a virtual network named myAKSVnet with the address prefix of 192.168.0.0/16 and a subnet named myAKSSubnet with the address prefix 192.168.1.0/24:

az network vnet create --resource-group $rgName --name $vnetName --address-prefixes $vnetAddPrefix --subnet-name $subnetName --subnet-prefix $subnetPrefix



https://azure.github.io/application-gateway-kubernetes-ingress/how-tos/networking/

Deployed in different vnets
AKS can be deployed in different virtual network from Application Gateway's virtual network, however, the two virtual networks must be peered together. When you create a virtual network peering between two virtual networks, a route is added by Azure for each address range within the address space of each virtual network a peering is created for.

aksClusterName="<aksClusterName>"
aksResourceGroup="<aksResourceGroup>"
appGatewayName="<appGatewayName>"
appGatewayResourceGroup="<appGatewayResourceGroup>"

# get aks vnet information
nodeResourceGroup=$(az aks show -n $aksClusterName -g $aksResourceGroup -o tsv --query "nodeResourceGroup")
aksVnetName=$(az network vnet list -g $nodeResourceGroup -o tsv --query "[0].name")
aksVnetId=$(az network vnet show -n $aksVnetName -g $nodeResourceGroup -o tsv --query "id")

# get gateway vnet information
appGatewaySubnetId=$(az network application-gateway show -n $appGatewayName -g $appGatewayResourceGroup -o tsv --query "gatewayIpConfigurations[0].subnet.id")
appGatewayVnetName=$(az network vnet show --ids $appGatewaySubnetId -o tsv --query "name")
appGatewayVnetId=$(az network vnet show --ids $appGatewaySubnetId -o tsv --query "id")

# set up bi-directional peering between aks and gateway vnet
az network vnet peering create -n gateway2aks \
    -g $appGatewayResourceGroup --vnet-name $appGatewayVnetName \
    --remote-vnet $aksVnetId \
    --allow-vnet-access
az network vnet peering create -n aks2gateway \
    -g $nodeResourceGroup --vnet-name $aksVnetName \
    --remote-vnet $appGatewayVnetId \
    --allow-vnet-access
If you are using Azure CNI for network plugin with AKS, then you are good to go.

If you are using Kubenet network plugin, then jump to Kubenet to setup the route table.

With Kubenet
When using Kubenet mode, Only nodes receive an IP address from subnet. Pod are assigned IP addresses from the PodIPCidr and a route table is created by AKS. This route table helps the packets destined for a POD IP reach the node which is hosting the pod.

When packets leave Application Gateway instances, Application Gateway's subnet need to aware of these routes setup by the AKS in the route table.

A simple way to achieve this is by associating the same route table created by AKS to the Application Gateway's subnet. 
When AGIC starts up, it checks the AKS node resource group for the existence of the route table. 
If it exists, AGIC will try to assign the route table to the Application Gateway's subnet, given it doesn't already have a route table. 
If AGIC doesn't have permissions to any of the above resources, the operation will fail and an error will be logged in the AGIC pod logs.

This association can also be performed manually:

aksClusterName="<aksClusterName>"
aksResourceGroup="<aksResourceGroup>"
appGatewayName="<appGatewayName>"
appGatewayResourceGroup="<appGatewayResourceGroup>"

# find route table used by aks cluster
nodeResourceGroup=$(az aks show -n $aksClusterName -g $aksResourceGroup -o tsv --query "nodeResourceGroup")
routeTableId=$(az network route-table list -g $nodeResourceGroup --query "[].id | [0]" -o tsv)

# get the application gateway's subnet
appGatewaySubnetId=$(az network application-gateway show -n $appGatewayName -g $appGatewayResourceGroup -o tsv --query "gatewayIpConfigurations[0].subnet.id")

# associate the route table to Application Gateway's subnet
az network vnet subnet update \
--ids $appGatewaySubnetId
--route-table $routeTableId




az aks list -g NEXUS -o table

az aks show \
    --resource-group NEXUS \
    --name nexus \
    --query apiServerAccessProfile.authorizedIpRanges





1. Create a resource group using the [az group create](https://learn.microsoft.com/en-us/cli/azure/group#az_group_create) command.
az group create --name $rgName --location $location

2. If you don't have an existing virtual network and subnet to use, create these network resources using the [az network vnet create](https://learn.microsoft.com/en-us/cli/azure/network/vnet#az_network_vnet_create) command. The following example command creates a virtual network named myAKSVnet with the address prefix of 192.168.0.0/16 and a subnet named myAKSSubnet with the address prefix 192.168.1.0/24:

az network vnet create --resource-group $rgName --name $vnetName --address-prefixes $vnetAddPrefix --subnet-name $subnetName --subnet-prefix $subnetPrefix

3. Get the subnet resource ID using the [az network vnet subnet show](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet#az_network_vnet_subnet_show) command and store it as a variable named SUBNET_ID for later use.

SUBNET_ID=$(az network vnet subnet show --resource-group $rgName --vnet-name $vnetName --name $subnetName --query id -o tsv)

4. Create AKS cluster

az aks create \
--resource-group $rgName \
--name $aksName \
--node-count 2 \
--max-pods 80 \
--network-plugin kubenet \
--vnet-subnet-id ${SUBNET_ID} \
--generate-ssh-keys -y \
--enable-managed-identity \
--node-vm-size $nodeVmSize \
--enable-cluster-autoscaler \
--min-count 1 \
--max-count 2 \
--pod-cidr 10.244.0.0/16 \
--debug


5. Update AKS Cluster Status

AKS_RESOURCE_ID=$(az aks show --name $aksName --resource-group $rgName --query 'id' -o tsv)

az resource update --ids ${AKS_RESOURCE_ID}

/subscriptions/3e2cab00-53cf-469c-a248-2f4271c79483/resourceGroups/Sherwin/providers/Microsoft.ContainerService/managedClusters/gg-sy-sh-sbx-01

AKS_RESOURCE_ID=$(az aks show --name gg-cls-app-01 --resource-group Synkrato-rg --query 'id' -o tsv)

AKS_RESOURCE_ID=$(az aks show --name gg-sy-za-app-01 --resource-group Synkrato-rg --query 'id' -o tsv)

az resource update --ids ${AKS_RESOURCE_ID}
