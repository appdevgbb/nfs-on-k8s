# nfs-on-k8s

## Cluster Setup

```bash
# Set Environment Variables
RG=AKSNFSLab
LOC=eastus
CLUSTER_NAME=nfslab
NODE_SKU=Standard_L16as_v3
ACR_NAME=nfslab
VNET_NAME=aksnfslab

# Create the Resource Group
az group create -n $RG -l $LOC

# Create Vnet and subnet
az network vnet create \
-g $RG \
-n $VNET_NAME \
--address-prefix 10.140.0.0/16 \
--subnet-name aks \
--subnet-prefix 10.140.0.0/24

# Get the cluster VNet Subnet ID
VNET_SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name $VNET_NAME -n aks -o tsv --query id)

# Create the Azure Container Registry
az acr create -g $RG -n $ACR_NAME --sku premium

# Create the AKS Cluster
az aks create -g $RG -n $CLUSTER_NAME \
--attach-acr $ACR_NAME \
--node-vm-size $NODE_SKU \
--enable-ultra-ssd \
--zones 1 \
--vnet-subnet-id $VNET_SUBNET_ID

# Get credentials
az aks get-credentials -g $RG -n $CLUSTER_NAME
```

## Deploy the Test Storage Class, PVC and Pod

```bash
kubectl apply -f manifests/ultra-ssd-sc.yaml
kubectl apply -f manifests/ultra-ssd-pvc.yaml
kubectl apply -f manifests/pod.yaml

kubectl exec -it ultra-test -- bash
apt update;apt install -y fio

cat << EOF > direct_device.fio
[global]
bs=1M
size=4G
iodepth=256
direct=1
ioengine=libaio
group_reporting
time_based
runtime=120
numjobs=1
                            
[raw-seq-write]
filename=/mnt/azure/test
rw=write

[raw-seq-read]
filename=/mnt/azure/test
rw=read

[raw-rand-write]
filename=/mnt/azure/test
rw=randwrite

[raw-rand-read]
filename=/mnt/azure/test
rw=randread
EOF

# Run the fio test
fio direct_device.fio
```
