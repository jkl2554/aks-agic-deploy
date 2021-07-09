[AGIC] https://docs.microsoft.com/ko-kr/azure/application-gateway/ingress-controller-overview
[AAD pod Identity] https://github.com/Azure/aad-pod-identity/blob/master/README.md


SAMPLE
```
### 변수 설정 ###

$SUB_ID="<구독 ID>"

$VNET_RG="<가상네트워크 RG>"
$VNET="<가상네트워크 명>"

$AGW_RG="<AGW RG>"
$AGW_NAME="<AGW 명>"
$AGW_SUBNET="<AGW subnet 명>"

$AKS_SERVICE_RG="<AKS RG>"
$AKS_SERVICE="<AKS 명>"
$AKS_SERVICE_SUBNET="<AKS subnet명>"

$INGRESS_CLASS="<AGW ingress Class>"

### account set
az account set -s $SUB_ID

### VNET 생성
az group create -n $VNET_RG -l Koreacentral
az network vnet create -n $VNET -g $VNET_RG --address-prefixes 10.0.0.0/16


### AKS 생성 ###
az group create -g $AKS_SERVICE_RG -l koreacentral

$AKS_SUBNET=az network vnet subnet create -g $VNET_RG -n $AKS_SERVICE_SUBNET --address-prefixes 10.0.1.0/24 --vnet-name $VNET | ConvertFrom-Json

az aks create -n $AKS_SERVICE -g $AKS_SERVICE_RG `
            --load-balancer-sku Standard --enable-rbac `
            --network-plugin azure --node-count 1 `
            --node-vm-size Standard_B2s --vnet-subnet-id $AKS_SUBNET.id `
            --service-cidr 10.255.0.0/24  --dns-service-ip 10.255.0.10 `
            --docker-bridge-address 10.255.1.1/24 --enable-managed-identity -y


$MC_AKS_RG=$(az aks show -n $AKS_SERVICE -g $AKS_SERVICE_RG --query "nodeResourceGroup" -o tsv)

### AGW 생성 ###

az group create -n $AGW_RG -l koreacentral

$AGW_SUBNET=az network vnet subnet create -n $AGW_SUBNET -g $VNET_RG --address-prefixes 10.0.0.0/24 --vnet-name $VNET |ConvertFrom-Json

az network application-gateway create -n $AGW_NAME -g $AGW_RG --sku Standard_v2 --public-ip-address $AGW_NAME-pip --subnet $AGW_SUBNET.id

## Managed Identity info
$MID=az identity show -n $AKS_SERVICE-agentpool -g $MC_AKS_RG | ConvertFrom-Json
$MID_ID=$MID.id
$MID_CLIENT_ID=$MID.clientId

# role assignment
az role assignment create --role Contributor --assignee $MID_CLIENT_ID --scope /subscriptions/$SUB_ID/resourceGroups/$AGW_RG/providers/Microsoft.Network/applicationGateways/$AGW_NAME
az role assignment create --role Reader --assignee $MID_CLIENT_ID --scope /subscriptions/$SUB_ID/resourceGroups/$AGW_RG
$aksVmssId=$(az vmss list -g $MC_AKS_RG --query "[0].id" -o tsv)
az role assignment create --role Reader --assignee $MID_CLIENT_ID --scope $aksVmssId

az aks get-credentials -n $AKS_SERVICE -g $AKS_SERVICE_RG

##############################

## pod-aad install
kubectl create ns pod-aad

helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
helm repo update
# Helm 3
helm install aad-pod-identity aad-pod-identity/aad-pod-identity -n pod-aad

## AGIC install
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update

kubectl create namespace agic

helm install agic application-gateway-kubernetes-ingress/ingress-azure `
     --namespace agic `
     --set appgw.name=$AGW_NAME `
     --set appgw.resourceGroup=$AGW_RG `
     --set appgw.subscriptionId=$SUB_ID `
     --set appgw.usePrivateIP=false `
     --set appgw.shared=false `
     --set armAuth.type=aadPodIdentity `
     --set armAuth.identityResourceID=$MID_ID `
     --set armAuth.identityClientID=$MID_CLIENT_ID `
     --set rbac.enabled=true `
     --set verbosityLevel=3 `
     --set kubernetes.ingressClass=$INGRESS_CLASS
```
