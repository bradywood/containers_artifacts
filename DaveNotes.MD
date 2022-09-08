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
export CLUSTER_NAME=tripAKSCluster
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
```
## List aks clusters
### az aks list -o table

## Assignment 4 - enable cluster with AD.

```
export AD_ADMIN_GROUP_ID=$(az ad group list --display-name openHackAKSAdminGroup --query "[0].id" --output tsv)

az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --network-plugin azure \
    --vnet-subnet-id $CLUSTER_SUBNET_ID \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip 10.2.2.10 \
    --service-cidr 10.2.2.0/24 \
    --enable-managed-identity --node-count 2 --enable-addons monitoring --enable-msi-auth-for-monitoring \
    --generate-ssh-keys --attach-acr registrytgz8460 \
    --enable-aad --aad-admin-group-object-ids $AD_ADMIN_GROUP_ID
```

### Need to reauthenticate to new cluster
```
az aks get-credentials --resource-group $openHackEastAsia_RG --name tripAKSCluster
```

### Create Namespace - kubectl create namespace web --dry-run=client -o yaml > yamls/namespace.yaml
```kubectl create -f yamls/namespace-web-api.yaml ```


### AD Group Create
```az ad group create --display-name openHackAKSAdminGroup --mail-nickname openHackAKSAdminGroup```
### LIST
```
export AD_ADMIN_GROUP_ID=$(az ad group list --display-name openHackAKSAdminGroup --query "[0].id" --output tsv)
```

### Create an AKS Cluster with AD enabled.. 
We need to go back and reacreate with the ad enabled.
az ad group list --filter "displayname eq '<group-name>'" -o table
az ad group create --display-name myAKSAdminGroup --mail-nickname myAKSAdminGroup
az aks create -g myResourceGroup -n myManagedCluster --enable-aad --aad-admin-group-object-ids $AD_ADMIN_GROUP_ID


### AD enable Cluster 
If for some reason AD was not enabled, run below.
```
az aks update -g $RESOURCE_GROUP -n $CLUSTER_NAME --enable-aad --aad-admin-group-object-ids $AD_ADMIN_GROUP_ID
```

### Create AD Groups
```
export AKS_CLUSTER_ID=$(az aks show \                                                    
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query id -o tsv)

export APIDEV_ID=$(az ad group create --display-name api-dev --mail-nickname api-dev --query id -o tsv)

export WEBDEV_ID=$(az ad group create --display-name web-dev --mail-nickname web-dev --query id -o tsv)
```

### To list
```
export APIDEV_ID=$(az ad group list --display-name api-dev --query "[0].id" --output tsv)
export WEBDEV_ID=$(az ad group list --display-name web-dev --query "[0].id" --output tsv)
```

### assign AD roles to the cluster
```
az role assignment create \
  --assignee $WEBDEV_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $AKS_CLUSTER_ID

az role assignment create \
  --assignee $APIDEV_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $AKS_CLUSTER_ID

```

### Create api and web users
```
AKSAPI_ID=$(az ad user create \
  --display-name "AKS API" \
  --user-principal-name "api@msftopenhack6742ops.onmicrosoft.com" \
  --password "Passw0rd" \
  --query "[0].id" --output tsv)

export AKSAPI_ID=$(az ad user list --upn "api@msftopenhack6742ops.onmicrosoft.com" --query "[0].id" -o tsv)

AKSWEB_ID=$(az ad user create \
  --display-name "AKS Web" \
  --user-principal-name "web@msftopenhack6742ops.onmicrosoft.com" \
  --password "Passw0rd" \
  --query "[0].id" --output tsv)

export AKSWEB_ID=$(az ad user list --upn "web@msftopenhack6742ops.onmicrosoft.com" --query "[0].id" -o tsv)

az ad group member add --group "api-dev" --member-id $AKSAPI_ID
az ad group member add --group "web-dev" --member-id $AKSWEB_ID
```  

### Create AKS Resource 
kubectl apply -f createrole.yaml

export API_GROUP_ID=$(az ad group show --group api-dev --query id -o tsv)
7a711655-809c-4c88-ac26-049ea401786b

export WEB_GROUP_ID=$(az ad group show --group web-dev --query id -o tsv)
04802ec4-50f5-4fe3-8584-9d26ec3e15e9

### Role Binding
```
kubectl apply -f yamls/rolebinding.yaml 
```


## Assignment 4 - NAMESpaces
### Services and namespaces
Some Notes
namespace.svc.cluster.local
http://trips.api.svc.cluster.local


### Reseting DB secrets.
```
kubectl create secret generic sqlsecret --from-literal=SQL_USER=sqladmintGz8460 --from-literal=SQL_PASSWORD=Passw0rd --dry-run=client -o yaml > ~/repos/azurehack/containers_artifacts/yamls/secret.yaml
```
Dont forget to use the namespace of API.

### Restarting pods
kubectl delete pod poi-7469f477c6-sjp4g -n api
kubectl delete pod trips-b45c7468d-bgwqv -n api
kubectl delete pod userprofile-965fcb9fd-h4jb8 -n api
kubectl delete pod userjava-9f5b97d9-lf7q9 -n api
