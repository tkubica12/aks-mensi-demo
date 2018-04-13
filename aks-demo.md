# AKS DEMO
## Vzdalena pracovni stanice
az group create -n tomasaksdemo -l westeurope
az vm create -n jumpserver -g tomasaksdemo \
    --admin-username tomas \
    --authentication-type password \
    --admin-password Azure12345678 \
    --image UbuntuLTS \
    --nsg "" \
    --public-ip-address-dns-name tomasjumpserver \
    --size Standard_DS1_v2 \
    --custom-data ./cloud-init.yaml \
    --storage-sku Premium_LRS

scp .secrets tomas@tomasjumpserver.westeurope.cloudapp.azure.com:~/
ssh tomas@tomasjumpserver.westeurope.cloudapp.azure.com


## AKS
az login

az aks create -n aks-demo \
    -g tomasaksdemo \
    --kubernetes-version 1.9.6 \
    --generate-ssh-keys \
    --node-count 3 \
    --node-vm-size Standard_DS1_v2

az aks get-credentials -n aks-demo -g tomasaksdemo

## ACR
az acr create -n tomasacr -g tomasaksdemo --sku Basic
az acr login -n tomasacr

docker build ./container/version1 --tag tomasacr.azurecr.io/mojeappka:1
docker build ./container/version2 --tag tomasacr.azurecr.io/mojeappka:2

docker run -p 3000:3000 -d tomasacr.azurecr.io/mojeappka:1
curl 127.0.0.1:3000

push ve VS Code

## Povolit login do ACR pro service principala (muze trvat i 15 minut)
AKS_RESOURCE_GROUP=tomasaksdemo
AKS_CLUSTER_NAME=aks-demo
ACR_RESOURCE_GROUP=tomasaksdemo
ACR_NAME=tomasacr
CLIENT_ID=$(az aks show --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" --output tsv)
ACR_ID=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "id" --output tsv)
az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

## Deployment, dostupnost, skalovani

kubectl apply -f deploymentWeb1.yaml
kubectl get pods -w
kubectl scale --replicas=5 -f deploymentWeb1.yaml
kubectl get pods -w

## Service
kubectl apply -f serviceWeb.yaml
kubectl apply -f podUbuntu.yaml

kubectl exec ubuntu -- curl -s mojeappka-service

## Externi pristup
kubectl get service -w
export extPublicIP=$(kubectl get service mojeappka-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl $extPublicIP

## Rolling upgrade
while true; do curl $extPublicIP; sleep 0.3; done
kubectl apply -f deploymentWeb2.yaml
kubectl get pods -w

## Ingress
helm init
helm install --name ingress stable/nginx-ingress \
    --set controller.nodeSelector="agentpool: nodepool1"

helm install stable/kube-lego --namespace kube-system \
    --name kube-lego \
    --set config.LEGO_EMAIL=tkubica@centrum.cz,config.LEGO_URL=https://acme-v01.api.letsencrypt.org/directory

kubectl get service -w
export ingressIP=$(kubectl get service ingress-nginx-ingress-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
az network dns record-set a delete -y -n app -g shared-services -z azure.tomaskubica.cz
az network dns record-set a add-record -a $ingressIP -n app -g shared-services -z azure.tomaskubica.cz

kubectl apply -f ingressWebLego.yaml
curl -v https://app.azure.tomaskubica.cz/

## ACI
az container create -n mssql \
    -g tomasaksdemo \
    --cpu 2 \
    --memory 4 \
    --ip-address public \
    --port 1433 -l westeurope \
    --image microsoft/mssql-server-linux \
    -e 'ACCEPT_EULA=Y' 'SA_PASSWORD=my(!)Password' 

az container logs -n mssql -g tomasaksdemo

## Virtual Kubelet
az aks install-connector -g tomasaksdemo \
    -n aks-demo \
    --os-type both \
    --connector-name myaciconnector

## Service Catalog

helm repo add svc-cat https://svc-catalog-charts.storage.googleapis.com
helm install svc-cat/catalog --name catalog --namespace services --set rbacEnable=false

source ~/.secrets

helm repo add azure https://kubernetescharts.blob.core.windows.net/azure
helm install azure/open-service-broker-azure --name azurebroker --namespace services \
  --set azure.subscriptionId=$subscription \
  --set azure.tenantId=$tenant \
  --set azure.clientId=$principal \
  --set azure.clientSecret=$client_secret \

svcat get brokers
svcat sync broker osba
svcat get classes
svcat describe class -t azure-postgresql
svcat describe class -t azure-cosmos-mongo-db
svcat describe class -t azure-mysql
svcat describe class -t azure-sql

kubectl apply -f serviceCatalogDemo.yaml

az sql server list -g myservices

kubectl exec env -- env | grep DB

## Helm + Service Catalog

helm repo add azure https://kubernetescharts.blob.core.windows.net/azure

helm install azure/wordpress --name wp \
    --set wordpressUsername=tomas \
    --set wordpressPassword=Azure12345678 \
    --set mysql.azure.location=westeurope 

echo http://$(kubectl get svc wp-wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/admin


## Draft

## Brigade ?