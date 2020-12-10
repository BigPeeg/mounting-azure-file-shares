# mounting-azure-file-shares
Step-by step example on how to setup a Kubenetes Deployment that mounts an Azure File Share

## Pre-requisites
Az CLI
Azure Subscription

## Steps
The following commands are entered at the command line (Powershell).

### Create an Azure file share

Create a storage account
```
$resourceGroupName="pg-icap-rg"
$storageAccountName="pgicapstorage"
$region="UK South"

az storage account create \
    --resource-group $resourceGroupName \
    --name $storageAccountName \
    --location $region \
    --kind StorageV2 \
    --sku Standard_LRS \
    --enable-large-file-share \
    --output none
```

Get the Storage Account Access Key
```
$storageAccountKey=$(az storage account keys list -g $resourceGroupName -n $storageAccountName --query [0].value)
```

Create an Azure File Share
```
$shareName="pgshare"

az storage share create \
    --account-name $storageAccountName \
    --account-key $storageAccountKey \
    --name $shareName \
    --quota 1024 \
    --output none

```

### Setup Kubernetes Storage
Create Fileshare Secret
```
$namespace="filesharetest"
kubectl create namespace $namespace

kubectl create secret generic azure-fileshare-secret --from-literal=azurestorageaccountname=$storageAccountName --from-literal=azurestorageaccountkey=$storageAccountKey -n $namespace
```

Create the Persistent Volume
```
kubectl apply -f pv.yaml
```

Create the Persistent Volume Claim
- the access mode must match that of the PV
- selector/matchLabels, "usage" value matches that of the PV labels/usage

```
kubectl apply -f pvc.yaml -n $namespace
```
Check that the components have been created
```
> kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                            STORAGECLASS      REASON   AGE
fileshare-pv                               10Gi       RWX            Retain           Bound      filesharetest/fileshare-pvc                                 6m29s

> kubectl get pvc -n $namespace
NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
fileshare-pvc   Bound    fileshare-pv   10Gi       RWX                           3m54s
```
Deploy the pods
```
kubectl apply -f fileshare-deployment.yaml -n $namespace
```

Check the deployment
```
> kubectl get pods -n $namespace
NAME                                    READY   STATUS    RESTARTS   AGE
fileshare-deployment-5d667df6fb-ffk6l   1/1     Running   0          23s
fileshare-deployment-5d667df6fb-fp96d   1/1     Running   0          23s
fileshare-deployment-5d667df6fb-pvzvq   1/1     Running   0          23s

```

### Check the Storage Mount

Exec into the first pod
```
kubectl exec -it fileshare-deployment-5d667df6fb-ffk6l -n $namespace -- sh
```
Navigate into the mounted folder and create some content.
```
/ # ls
bin          dev          home         root         tmp          var
configfiles  etc          proc         sys          usr

/ # cd configfiles
/configfiles # echo "Hello from first pod" >> pod1.txt
/configfiles # ls
pod1.txt
/configfiles # exit

```
Exec into another pod and check the content is there
```
kubectl exec -it fileshare-deployment-5d667df6fb-fp96d -n $namespace -- sh

/ # cd configfiles/
/configfiles # ls
pod1.txt
/configfiles # cat pod1.txt
Hello from first pod
/configfiles # exit
```

### Delete and Redeploy

```
> kubectl delete -f fileshare-deployment.yaml -n $namespace
deployment.apps "fileshare-deployment" deleted
> kubectl apply -f fileshare-deployment.yaml -n $namespace
deployment.apps/fileshare-deployment created
```
Check the pods have restarted
```
> kubectl get pods -n $namespace
NAME                                    READY   STATUS    RESTARTS   AGE
fileshare-deployment-5d667df6fb-cchj6   1/1     Running   0          60s
fileshare-deployment-5d667df6fb-wrpz5   1/1     Running   0          60s
fileshare-deployment-5d667df6fb-wtwg7   1/1     Running   0          60s
```
Exec into a pod and check the content is there
```
kubectl get pods -n $namespace
NAME                                    READY   STATUS    RESTARTS   AGE
fileshare-deployment-5d667df6fb-cchj6   1/1     Running   0          60s
fileshare-deployment-5d667df6fb-wrpz5   1/1     Running   0          60s
fileshare-deployment-5d667df6fb-wtwg7   1/1     Running   0          60s

> kubectl exec -it fileshare-deployment-5d667df6fb-wtwg7 -n $namespace -- sh
/ # cd configfiles/
/configfiles # ls
pod1.txt
/configfiles # cat pod1.txt
Hello from first pod
/configfiles # exit

```
