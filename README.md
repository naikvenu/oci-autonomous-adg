# oci-autonomous-adg

This repository holds the steps to perform Autonomous database disaster recovery using Mushop Demo application

The following steps would perform a DR solution between ap-mumbai-1 and ap-hyderabad-1 regions.

# Set Environment variables
```
export COMPARTMENT_ID=<Your_compartment_id>
export DB_NAME=demoadb
export DB_DISPLAY_NAME=demoadb
export DB_PASSWORD=DemoAdb#1234
export WALLET_PW="DemoAdb#1234"
export WALLET_ZIP="/tmp/Wallet_${DB_NAME}.zip"
```

# Create the Source DB (region ap-mumbai-1)

``$ oci db autonomous-database  create --compartment-id ${COMPARTMENT_ID} --db-name ${DB_NAME} --admin-password ${DB_PASSWORD} --db-version 19c --cpu-core-count 1 --data-storage-size-in-tbs 1 --display-name ${DB_DISPLAY_NAME}``

## Get the Source DB OCID:

``DB_ID=$(oci db autonomous-database list -c ${COMPARTMENT_ID} --query "data[?\"db-name\"=='${DB_NAME}'].id | [0]" --raw-output)``

## Create the DR (Creating in region ap-hyderabad-1):

``$ oci db autonomous-database  create-adb-cross-region-data-guard-details  --compartment-id ${COMPARTMENT_ID} --db-name ${DB_NAME} --source-id ${DB_ID} --cpu-core-count 1 --data-storage-size-in-tbs 1 --region ap-hyderabad-1 --db-version 19c``

## Download the wallet:

``$ oci db autonomous-database generate-wallet --autonomous-database-id ${DB_ID} --password ${WALLET_PW} --file ${WALLET_ZIP}``

- Extract Wallet:
``$ unzip Wallet_demoadb.zip -d wallet_source``

**Note** the database wallet has to be separately downloaded for primary and remote region's as tnsnames.ora DNS entries are slightly different.

# Mushop Setup (Source):

## Pre-Requisites:

This procedure assumes you have created a OKE (Oracle Cloud Infrastructure Container Engine for Kubernetes) cluster using this 
guide: https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-full/index.html#DefineClusterDetails
	
	
The Mushop setup instructions are taken from: https://github.com/oracle-quickstart/oci-cloudnative/tree/master/deploy/complete/helm-chart#setup and tested as below:

``$ git clone https://github.com/oracle-quickstart/oci-cloudnative.git

$ cd oci-cloudnative/deploy/complete/helm-chart``

## Update chart dependencies:

``$ helm dependency update ./setup``

## Install setup chart:

``$ helm install mushop-utils setup --namespace mushop-utilities --create-namespace

$ kubectl create namespace mushop``

## For this below step of adding secrets, do note to add ap-mumbai-1 for source and ap-hyderabad-1 for DR in our case. 

## The source region in this case would be ap-mumbai-1

```$ kubectl create secret generic oci-credentials \
  --namespace mushop \
  --from-literal=tenancy=<TENANCY_OCID> \
  --from-literal=user=<USER_OCID> \
  --from-literal=region=<USER_OCI_REGION> \
  --from-literal=fingerprint=<USER_PUBLIC_API_KEY_FINGERPRINT> \
  --from-literal=passphrase=<PASSPHRASE_STRING> \
  --from-file=privatekey=<PATH_OF_USER_PRIVATE_API_KEY>
  
$ kubectl create secret generic oadb-admin \
  --namespace mushop \
  --from-literal=oadb_admin_pw='DemoAdb#1234'
  
$ kubectl create secret generic oadb-wallet   --namespace mushop   --from-file=/tmp/wallet_source
  
$ kubectl create secret generic oadb-connection \
  --namespace mushop \
  --from-literal=oadb_wallet_pw='DemoAdb#1234' \
  --from-literal=oadb_service='demoadb_tp'
 ```
  
## Edit/Add the following secrets to values-prod.yaml as shown below:
  
```$ cat mushop/values-prod.yaml
global:
  ociAuthSecret: oci-credentials        # OCI authentication credentials secret name
  ossStreamSecret:                      # Name of Stream Connection secret
  oadbAdminSecret: oadb-admin           # Name of DB Admin secret created earlier
  oadbWalletSecret: oadb-wallet         # Name of wallet secret created earlier
  oadbConnectionSecret: oadb-connection # Name of connection secret created earlier

$ helm install -f ./mushop/values-prod.yaml mymushop mushop -n mushop
```

## Perform Autonomous transaction Processing (ATP) Failover

Go to OCI console and perform a failover.

# MuShop Setup (DR):

Follow all the steps from MuShop Setup (Source), the changing steps are highlighted

## The DR region would be ap-hyderabad-1:

```$ kubectl create secret generic oci-credentials \
  --namespace mushop \
  --from-literal=tenancy=<TENANCY_OCID> \
  --from-literal=user=<USER_OCID> \
  --from-literal=region=<USER_OCI_REGION> \
  --from-literal=fingerprint=<USER_PUBLIC_API_KEY_FINGERPRINT> \
  --from-literal=passphrase=<PASSPHRASE_STRING> \
  --from-file=privatekey=<PATH_OF_USER_PRIVATE_API_KEY>
```
  
## Command for wallet would change for the DR:

``$ kubectl create secret generic oadb-wallet   --namespace mushop   --from-file=/tmp/wallet_remote``

``$ kubectl create secret generic oadb-admin \
  --namespace mushop \
  --from-literal=oadb_admin_pw='DemoAdb#1234'
  
$ kubectl create secret generic oadb-wallet   --namespace mushop   --from-file=/tmp/wallet_source
  
$ kubectl create secret generic oadb-connection \
  --namespace mushop \
  --from-literal=oadb_wallet_pw='DemoAdb#1234' \
  --from-literal=oadb_service='demoadb_tp'
  
$ cat mushop/values-prod.yaml
global:
  ociAuthSecret: oci-credentials        # OCI authentication credentials secret name
  ossStreamSecret:                      # Name of Stream Connection secret
  oadbAdminSecret: oadb-admin           # Name of DB Admin secret created earlier
  oadbWalletSecret: oadb-wallet         # Name of wallet secret created earlier
  oadbConnectionSecret: oadb-connection # Name of connection secret created earlier

$ helm install -f ./mushop/values-prod.yaml mymushop mushop -n mushop

``
  
  
## Setting up the ingress :

**Note**. The below ingress instructions has to be performed on both regions on the OKE nodes (ap-mumbai-1 and ap-hyderabad-1) :

```openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginxsvc/O=nginxsvc"
kubectl create secret tls tls-secret --key tls.key --cert tls.crt -n mushop

 cat << EOF | kubectl -n mushop apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: mushop
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - secretName: tls-secret
  rules:
  - http:
      paths:
      - backend:
          serviceName: edge
          servicePort: 80
EOF
```


  
# Access the cluster using the ingress IP:

``$ kubectl get svc mushop-utils-ingress-nginx-controller -n mushop-utilities``

# Testing
  
Note: Ensure that you have done the above steps on both the regions and can access the Mushop Page. For DR system the Mushop products would
not be shown and a Error 500 would be displayed and this is because the DB on the DR site is on standby.


