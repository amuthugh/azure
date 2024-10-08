AKS Instance creation in d16a7c0e-f05a-4b10-a60f-72f7c3843111

1. Make sure you have subscription details to connect
2. Create Resource group
3. Create VNet and Subnet
4. Create AKS cluster
5. Connect MongoDB through Vnet peering

**Create a virtual network and subnet**
```
az account set --subscription d16a7c0e-f05a-4b10-a60f-72f7c3843111
export aksName=gg-ex-rnd-db-01
export rgName=gg-ex-rnd-central-us-rg
export location=centralus 
export vnetName=gg-ex-rnd-central-us-vnet
export vnetAddPrefix=10.9.16.0/20
export subnetName=gg-ex-rnd-db-01-subnet
export subnetPrefix=10.9.16.0/26
export nodeVmSize=Standard_D2as_v5
export tagsSn="Product=Exto Customer=exto Environment=Non-Prod HasNexus=No Applications=Exto-platform,Bolo,Postgres"

```

1. Create a resource group using the [az group create](https://learn.microsoft.com/en-us/cli/azure/group#az_group_create) command. Location name can be find here :  https://gist.github.com/ausfestivus/04e55c7d80229069bf3bc75870630ec8
```
az group create --name $rgName --location $location --tags $tagsSn
```
2. If you don't have an existing virtual network and subnet to use, create these network resources using the [az network vnet create](https://learn.microsoft.com/en-us/cli/azure/network/vnet#az_network_vnet_create) command. The following example command creates a virtual network named myAKSVnet with the address prefix of 192.168.0.0/16 and a subnet named myAKSSubnet with the address prefix 192.168.1.0/24:

```
az network vnet create --resource-group $rgName --name $vnetName --address-prefixes $vnetAddPrefix --subnet-name $subnetName --subnet-prefix $subnetPrefix --tags $tagsSn

```

3. Get the subnet resource ID using the [az network vnet subnet show](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet#az_network_vnet_subnet_show) command and store it as a variable named SUBNET_ID for later use.

```
SUBNET_ID=$(az network vnet subnet show --resource-group $rgName --vnet-name $vnetName --name $subnetName --query id -o tsv)
```
4. Create AKS cluster
```
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
--debug \
--tags $tagsSn
```

5. Update AKS Cluster Status

AKS_RESOURCE_ID=$(az aks show --name $aksName --resource-group $rgName --query 'id' -o tsv)

az resource update --ids ${AKS_RESOURCE_ID}

/subscriptions/####/resourceGroups/Sherwin/providers/Microsoft.ContainerService/managedClusters/gg-sy-sh-sbx-01

AKS_RESOURCE_ID=$(az aks show --name gg-cls-app-01 --resource-group Synkrato-rg --query 'id' -o tsv)

AKS_RESOURCE_ID=$(az aks show --name gg-sy-za-app-01 --resource-group Synkrato-rg --query 'id' -o tsv)

az resource update --ids ${AKS_RESOURCE_ID}


cli.azure.cli.core.sdk.policies: {"error":{"code":"PrincipalNotFound","message":"Principal c3556a780c1d48efa7537ecfc2ec854d does not exist in the directory 6f292041-81af-4175-8575-985683babb87. Check that you have the correct principal ID. If you are creating this principal and then immediately assigning a role, this error might be related to a replication delay. In this case, set the role assignment principalType property to a value, such as ServicePrincipal, User, or Group.  See https://aka.ms/docs-principaltype"}}
cli.azext_aks_preview._roleassignments: (PrincipalNotFound) Principal c3556a780c1d48efa7537ecfc2ec854d does not exist in the directory 6f292041-81af-4175-8575-985683babb87. Check that you have the correct principal ID. If you are creating this principal and then immediately assigning a role, this error might be related to a replication delay. In this case, set the role assignment principalType property to a value, such as ServicePrincipal, User, or Group.  See https://aka.ms/docs-principaltype
Code: PrincipalNotFound
Message: Principal c3556a780c1d48efa7537ecfc2ec854d does not exist in the directory 6f292041-81af-4175-8575-985683babb87. Check that you have the correct principal ID. If you are creating this principal and then immediately assigning a role, this error might be related to a replication delay. In this case, set the role assignment principalType property to a value, such as ServicePrincipal, User, or Group.  See https://aka.ms/docs-principaltype
