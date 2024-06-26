<h1 align="center">
  <br>
  Aks and Application Gateway Ingress Controller configration
  <br>
</h1>

# Step by step
1. [Create Application gateway](/AzureApplicationGateway.md#1-create-application-gateway)
2. [Create AKS Cluster](/AzureApplicationGateway.md#2-create-aks-cluster)
3. [Configure AGIC configuration](/AzureApplicationGateway.md#3-configure-agic-in-aks)

### 1. Create Application gateway

*** Application Gateway Variables ***

```
export pipName=intel-dev-pip
export appgwRgName=DevAgw-rg
export appgwName=DevAGW
export appgwVnetName=dev-appgw-vnet
export appgwSnetName=dev-appgw-snet-01
export location=centralindia
export appgwVnetPrefix=10.9.0.0/24
export appgwSnetPrefix=10.9.0.0/26

#Time being ignore : --waf-policy $wafPolicyName --sku WAF_v2 while creation
#export wafPolicyName=dev-waf-policy

```

## 1.1. Resource group, Vnet and Subnet creation for Application Gateway

### 1.1.1. Create a resource group using the [az group create](https://learn.microsoft.com/en-us/cli/azure/group#az_group_create) command.

```
az group create --name $appgwRgName --location $location

```

### 1.1.2. Create Vnet 
If you don't have an existing virtual network and subnet to use, create these network resources using the [az network vnet create](https://learn.microsoft.com/en-us/cli/azure/network/vnet#az_network_vnet_create) command. The following example command creates a virtual network named myAKSVnet with the address prefix of 192.168.0.0/16 and a subnet named myAKSSubnet with the address prefix 192.168.1.0/24:

```
az network vnet create --resource-group $appgwRgName --name $appgwVnetName --address-prefixes $appgwVnetPrefix --subnet-name $appgwSnetName --subnet-prefix $appgwSnetPrefix

```
Ignore : 1.1.3 Create WAF policy

```
Ignore : az network application-gateway waf-policy create --name $wafPolicyName --resource-group $appgwRgName
```
### 1.1.4 Create public ip
```
az network public-ip create -n $pipName -g $appgwRgName -l $location --allocation-method Static --sku Standard
```

### 1.1.5. Create application gateway
```
az network application-gateway create -n $appgwName -l $location -g $appgwRgName --sku Standard_v2 --public-ip-address $pipName --vnet-name $appgwVnetName --subnet $appgwSnetName --priority 100 
```

### 1.1.6 Azure Application Gateway Stop & Start
```
export appgwRgName=DevAgw-rg
export appgwName=DevAGW
```

```
az network application-gateway stop --name $appgwName --resource-group $appgwRgName
```
```
az network application-gateway start --name $appgwName --resource-group $appgwRgName
```
# 2. Create AKS Cluster

### 2.1 Resource group, Vnet and Subnet creation for AKS

*** AKS Variables ***
```
export aksRgName=IntelDevAks-rg
export aksName=gg-il-app01
export aksVnetName=gg-centralindia-vnet
export aksSubnetName=gg-il-app01-subnet-01
export location=centralindia
export aksVnetAddPrefix=10.10.0.0/1
export aksSubnetPrefix=10.10.0.0/26
export nodeVmSize=Standard_B2ms
```


2.1.1 Create a resource group using the [az group create](https://learn.microsoft.com/en-us/cli/azure/group#az_group_create) command.
```
az group create --name $aksRgName --location $location
```
2.1.2 If you don't have an existing virtual network and subnet to use, create these network resources using the [az network vnet create](https://learn.microsoft.com/en-us/cli/azure/network/vnet#az_network_vnet_create) command. The following example command creates a virtual network named myAKSVnet with the address prefix of 192.168.0.0/16 and a subnet named myAKSSubnet with the address prefix 192.168.1.0/24:
```
az network vnet create --resource-group $aksRgName --name $aksVnetName --address-prefixes $aksVnetAddPrefix --subnet-name $aksSubnetName --subnet-prefix $aksSubnetPrefix
```
https://azure.github.io/application-gateway-kubernetes-ingress/how-tos/networking/

### 2.1.3 VNet Peering

Deployed in different vnets
AKS can be deployed in different virtual network from Application Gateway's virtual network, however, the two virtual networks must be peered together. When you create a virtual network peering between two virtual networks, a route is added by Azure for each address range within the address space of each virtual network a peering is created for.

Create vnet peering APPGateway to AKS 
```
aksVnetId=$(az network vnet show -n $aksVnetName -g $aksRgName -o tsv --query "id")
echo $aksVnetId
az network vnet peering create -n AppGWtoAKSVnetPeering -g $appgwRgName --vnet-name $appgwVnetName --remote-vnet $aksVnetId --allow-vnet-access
```
Create vnet peering AKS to APPGateway

```
appGWVnetId=$(az network vnet show -n $appgwVnetName -g $appgwRgName -o tsv --query "id")
echo $appGWVnetId
az network vnet peering create -n AKStoAppGWVnetPeering -g $aksRgName --vnet-name $aksVnetName --remote-vnet $appGWVnetId --allow-vnet-access
```
### 2.2.1 Get the subnet resource ID using the [az network vnet subnet show](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet#az_network_vnet_subnet_show) command and store it as a variable named aksSubnetId for later use.

```
aksSubnetId=$(az network vnet subnet show --resource-group $aksRgName --vnet-name $aksVnetName --name $aksSubnetName --query id -o tsv)

echo $aksSubnetId
```

### 2.2.2 Create AKS cluster

```
az aks create \
--resource-group $aksRgName \
--name $aksName \
--node-count 1 \
--max-pods 80 \
--network-plugin kubenet \
--vnet-subnet-id ${aksSubnetId} \
--generate-ssh-keys -y \
--enable-managed-identity \
--node-vm-size $nodeVmSize \
--enable-cluster-autoscaler \
--min-count 1 \
--max-count 2 \
--pod-cidr 10.244.0.0/16 \
--debug
```
### 2.3 Start and Stop AKS Cluster
```
export aksRgName=IntelDevAks-rg
export aksName=gg-il-app01
```
```
az aks stop --name $aksName --resource-group $aksRgName
```
```
az aks start --name $aksName --resource-group $aksRgName
```

# 3. Configure AGIC in AKS

#### Important

When you use an application gateway in a different resource group than the AKS cluster resource group, the managed identity ingressapplicationgateway-{AKSNAME} that is created must have Contributor and Reader roles set in the application gateway resource group.


###  3.1 Enable Application Gateway Ingress Controller on AKS
```
export appgwRgName=DevAgw-rg
export appgwName=DevAGW
```

```
appgwId=$(az network application-gateway show -n $appgwName -g $appgwRgName -o tsv --query "id")
echo $appgwId
```
```
az aks enable-addons -n $aksName -g $aksRgName -a ingress-appgw --appgw-id $appgwId
```

### 3.2 Assign network contributor role to AGIC addon Managed Identity

```
export aksRgName=IntelDevAks-rg 
export aksName=gg-il-app01
export appgwRgName=DevAgw-rg
export appgwName=DevAGW
```
Get application gateway id from AKS addon profile
```
appGatewayId=$(az aks show -n $aksName -g $aksRgName -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")

echo $appGatewayId
```
Get Application Gateway subnet id
```
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")

echo $appGatewaySubnetId
```

Get AGIC addon identity
```
agicAddonIdentity=$(az aks show -n $aksName -g $aksRgName -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")

echo $agicAddonIdentity

```
Assign network contributor role to AGIC addon identity to subnet that contains the Application Gateway
```
az role assignment create --assignee $agicAddonIdentity --scope $appGatewaySubnetId --role "Network Contributor"
```

### 3.3 Associate the route table to Application Gateway's subnet

With Kubenet
When using Kubenet mode, Only nodes receive an IP address from subnet. Pod are assigned IP addresses from the PodIPCidr and a route table is created by AKS. This route table helps the packets destined for a POD IP reach the node which is hosting the pod.

When packets leave Application Gateway instances, Application Gateway's subnet need to aware of these routes setup by the AKS in the route table.

A simple way to achieve this is by associating the same route table created by AKS to the Application Gateway's subnet. 
When AGIC starts up, it checks the AKS node resource group for the existence of the route table. 
If it exists, AGIC will try to assign the route table to the Application Gateway's subnet, given it doesn't already have a route table. 
If AGIC doesn't have permissions to any of the above resources, the operation will fail and an error will be logged in the AGIC pod logs.

This association can also be performed manually:
```
export aksRgName=IntelDevAks-rg 
export aksName=gg-il-app01
export appgwRgName=DevAgw-rg
export appgwName=DevAGW
```
 Find route table used by aks cluster
```
nodeResourceGroup=$(az aks show -n $aksName -g $aksRgName -o tsv --query "nodeResourceGroup")
echo $nodeResourceGroup
```
```
routeTableId=$(az network route-table list -g $nodeResourceGroup --query "[].id | [0]" -o tsv)
echo $routeTableId
```
Get the application gateway's subnet
```
### Not fetching the ID Ignore
appGatewaySubnetId=$(az network application-gateway show -n $appgwName -g $appgwRgName -o tsv --query "gatewayIpConfigurations[0].subnet.id")
echo $appGatewaySubnetId
```
```
appGatewayId=$(az aks show -n $aksName -g $aksRgName -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")

echo $appGatewayId
```
```
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")

echo $appGatewaySubnetId
```

Associate the route table to Application Gateway's subnet
```
az network vnet subnet update --ids $appGatewaySubnetId --route-table $routeTableId
```

Referances : 

1)https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-existing
2)https://www.linkedin.com/pulse/create-aks-cluster-application-gateway-agic-using-external-chandio-zjkuf/

Verify the Configurations

kubectl config get-clusters
kubectl config get-contexts
kubectl config get-users
kubectl config view 

kubectl cluster-info
kubectl get cs

kubectl get nodes 
kubectl get nodes  -o wide

### Verify AKS Add On

# List Kubernetes Deployments in kube-system namespace
kubectl get deploy -n kube-system
Observation:
1. Should find the deployment with name "ingress-appgw-deployment"
2. This is the Azure Application Gateway Ingress Controller Kubernetes Deployment Object

# List Pods
kubectl get pods -n kube-system

# Describe Pod
kubectl -n kube-system describe pod <AGIC-POD-NAME>
kubectl -n kube-system describe pod ingress-appgw-deployment-55965f45cf-x28fm  
Observation:
1. Review the line where you can find ingress current version downloaded
2. Pulling image "mcr.microsoft.com/azure-application-gateway/kubernetes-ingress:1.5.3"
3. You can also run the below command to find the AppGW Ingress version
kubectl get deploy ingress-appgw-deployment -o yaml -n kube-system | grep "image:"

# Verify ingress-appgw pod Logs
kubectl -n kube-system logs -f $(kubectl -n kube-system get po | egrep -o 'ingress-appgw-deployment[A-Za-z0-9-]+')


Other usefull commands :

az aks list -g $aksRgName -o table

az aks show -n $aksName -g $aksRgName --query apiServerAccessProfile.authorizedIpRanges

Update AKS Cluster Status

AKS_RESOURCE_ID=$(az aks show --name $aksName --resource-group $aksRgName --query 'id' -o tsv)

az resource update --ids ${AKS_RESOURCE_ID}

