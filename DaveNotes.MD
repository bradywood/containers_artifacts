# Dave notes

## Key Commands
## az configure
configures the az to do things like set the standard table output.

## Containers 2.0 Openhack
## Assignment 1
```
az login
az acr login -n registrytgz8460

docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=Passw0rd" -p 1433:1433 --name sql1 --hostname sql1 -d mcr.microsoft.com/mssql/server:2017-latest

docker network create mylocal
docker network connect mylocal sql1

docker run --network mylocal -e SQLFQDN=sql1 -e SQLUSER=SA -e SQLPASS=Passw0rd -e SQLDB=mydrivingDB registrytgz8460.azurecr.io/dataload:1.0

export SQL_PASSWORD=Passw0rd
export SQL_SERVER=sql1
export ASPNETCORE_ENVIRONMENT=Local

docker run --network mylocal -d -p 8080:80 --name poi -e "SQL_USER=SA" -e "SQL_PASSWORD=$SQL_PASSWORD" -e "SQL_SERVER=$SQL_SERVER" -e "ASPNETCORE_ENVIRONMENT=Local" tripinsights/poi:1.0

docker run --network mylocal -d -p 8081:80 --name trips -e "SQL_USER=SA" -e "SQL_PASSWORD=$SQL_PASSWORD" -e "SQL_SERVER=$SQL_SERVER" -e "OPENAPI_DOCS_URI=http://$EXTERNAL_IP" tripinsights/trips:1.0
```
## Assignment 2
```
kubectl port-forward tripviewer-7bdff87d6d-hw9gx 8888:80
kubectl port-forward poi-7469f477c6-jtgr6 8889:80 &
kubectl port-forward trips-b45c7468d-w7p9r 8890:80 &
kubectl port-forward userjava-9f5b97d9-hvx5l 8891:80 &
```
## Assignment 3
```
az network vnet list -o table

export RESOURCE_GROUP=openHackEastAsia_RG
export VNET=vnet
export CLUSTER_SUBNET_NAME=tripAKSSubnet
export CLUSTER_ADDRESS_SPACE=10.2.1.0/24
export VNET_RESOURCE_GROUP=teamResources

export CLUSTER_SUBNET_ID=$(az network vnet subnet list \
    --resource-group teamResources \
    --vnet-name $VNET \
    --query "[0].id" --output tsv)

### if you wish to create..
export CLUSTER_SUBNET_ID=$(az network vnet subnet create \
    -g $VNET_RESOURCE_GROUP --vnet-name $VNET \
    -n $CLUSTER_SUBNET_NAME \
    --address-prefixes $CLUSTER_ADDRESS_SPACE \
    --query id -o tsv)

## List aks clusters
### az aks list -o table

az aks create \
    --resource-group $RESOURCE_GROUP \
    --name tripAKSCluster \
    --network-plugin azure \
    --vnet-subnet-id $CLUSTER_SUBNET_ID \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip 10.2.2.10 \
    --service-cidr 10.2.2.0/24 \
    --enable-managed-identity --node-count 2 --enable-addons monitoring --enable-msi-auth-for-monitoring \
    --generate-ssh-keys --attach-acr registrytgz8460 
```
```
### Need to reauthenticate to new cluster
```
az aks get-credentials --resource-group $openHackEastAsia_RG --name tripAKSCluster

## Create Namespace
### kubectl create namespace web --dry-run=client -o yaml > yamls/namespace.yaml
```kubectl create -f yamls/namespace-web-api.yaml ```


## Create an AKS Cluster with AD enabled.. 
We need to go back and reacreate with the ad enabled.
az ad group list --filter "displayname eq '<group-name>'" -o table
az aks create -g myResourceGroup -n myManagedCluster --enable-aad --aad-admin-group-object-ids <id> [--aad-tenant-id <id>]
