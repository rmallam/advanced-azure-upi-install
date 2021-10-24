# Pre Requisites

## Command line utilities's

az - Azure cli

yq - Yaml processor

openshift-install binary
Download installation program from cloud.redhat.com --> Clusters --> new cluster

# azure credentials
az login

az ad sp create-for-rbac --role Contributor --name azureupi

# get appid from the previous command and replace here
az role assignment create --role "User Access Administrator" --assignee-object-id $(az ad sp list --filter "appId eq '<appId>'" | jq '.[0].objectId' -r)

az ad app permission add --id <appId> --api 00000002-0000-0000-c000-000000000000 --api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role

az ad app permission grant --id <appId> --api 00000002-0000-0000-c000-000000000000


# Download installation program from cloud.redhat.com --> Clusters --> new cluster

export installdir=azureocp1

mkdir $installdir

# Create install config
openshift-install create install-config --dir=${installdir}

## Note: to use OVN network type, update install-config.yaml , Change networkType: from openshiftSDN to OVNKubernetes

export CLUSTER_NAME=`yq e '.metadata.name' ${installdir}/install-config.yaml`
export AZURE_REGION=`yq e '.platform.azure.region' ${installdir}/install-config.yaml`
export SSH_KEY=`yq e '.sshKey' ${installdir}/install-config.yaml`
export BASE_DOMAIN=`yq e '.baseDomain' ${installdir}/install-config.yaml`
export BASE_DOMAIN_RESOURCE_GROUP=`yq e '.platform.azure.baseDomainResourceGroupName'  ${installdir}/install-config.yaml`


echo $CLUSTER_NAME "\n"$AZURE_REGION "\n"$SSH_KEY "\n"$BASE_DOMAIN  "\n"$BASE_DOMAIN_RESOURCE_GROUP
export KUBECONFIG=${installdir}/auth/kubeconfig   

# Create Kube manifest files
openshift-install create manifests --dir=${installdir}  

# To enable ipsec with OVNKubernetes -- This cant be performed after cluster install

cat <<EOF > ${installdir}/manifests/cluster-network-03-config.yml
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  defaultNetwork:
    type: OVNKubernetes
    ovnKubernetesConfig:
      ipsecConfig: {}
      mtu: 1400
EOF

rm -rf ${installdir}/openshift/99_openshift-cluster-api_master-machines-*.yaml 
rm -rf ${installdir}/openshift/99_openshift-cluster-api_worker-machineset-*.yaml


export INFRA_ID=`grep infrastructureName  ${installdir}/manifests/cluster-infrastructure-02-config.yml | awk -F ':' '{print $2}'  | xargs `

export RESOURCE_GROUP=`grep resourceGroupName  ${installdir}/manifests/cluster-infrastructure-02-config.yml | awk -F ':' '{print $2}'  | xargs`

echo $INFRA_ID

echo $RESOURCE_GROUP

# Create ignition files
openshift-install create ignition-configs --dir ${installdir} 

# create Resource Group in azure to hold the resources
az group create --name ${RESOURCE_GROUP} --location ${AZURE_REGION}

# create identity 

az identity create -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity

export PRINCIPAL_ID=`az identity show -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity --query principalId --out tsv`

echo $PRINCIPAL_ID

export RESOURCE_GROUP_ID=`az group show -g ${RESOURCE_GROUP} --query id --out tsv`

echo $RESOURCE_GROUP_ID

az role assignment create --assignee "${PRINCIPAL_ID}" --role 'Contributor' --scope "${RESOURCE_GROUP_ID}"

# Create an Azure storage account to store the VHD cluster image
az storage account create -g ${RESOURCE_GROUP} --location ${AZURE_REGION} --name ${CLUSTER_NAME}sa --kind Storage --sku Standard_LRS

# get account key from storage account

export ACCOUNT_KEY=`az storage account keys list -g ${RESOURCE_GROUP} --account-name ${CLUSTER_NAME}sa --query "[0].value" -o tsv`

echo $ACCOUNT_KEY
export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.8/data/data/rhcos.json | jq -r .azure.url`

echo $VHD_URL
az storage container create --name vhd --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY}

az storage blob copy start --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "${VHD_URL}"

tstatus="unknown"
while [ "$tstatus" != "success" ]
do
  tstatus=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -o tsv --query properties.copy.status`
  echo $tstatus
done

# store ign files in azure blob

az storage container create --name files --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} --public-access blob

az storage blob upload --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c "files" -f "${installdir}/bootstrap.ign" -n "bootstrap.ign"

 # Create public and private dns zones

az network dns zone create -g ${BASE_DOMAIN_RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}

az network private-dns zone create -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}


# Create VNETs 

az deployment group create -g ${RESOURCE_GROUP} --template-file "${installdir}/01_vnet.json" --parameters baseName="${INFRA_ID}"

az network private-dns link vnet create -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n ${INFRA_ID}-network-link -v "${INFRA_ID}-vnet" -e false

# Creae deployment group for storage

export VHD_BLOB_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c vhd -n "rhcos.vhd" -o tsv`
echo $VHD_BLOB_URL

az deployment group create -g ${RESOURCE_GROUP} --template-file "${installdir}/02_storage.json" --parameters vhdBlobURL="${VHD_BLOB_URL}" --parameters baseName="${INFRA_ID}" 

# create networking and loadbalancing components
az deployment group create -g ${RESOURCE_GROUP} --template-file "${installdir}/03_infra.json" --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" --parameters baseName="${INFRA_ID}" 


export PUBLIC_IP=`az network public-ip list -g ${RESOURCE_GROUP} --query "[?name=='${INFRA_ID}-master-pip'] | [0].ipAddress" -o tsv`
echo $PUBLIC_IP

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n api -a ${PUBLIC_IP} --ttl 60

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${BASE_DOMAIN} -n api.${CLUSTER_NAME} -a ${PUBLIC_IP} --ttl 60

export BOOTSTRAP_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c "files" -n "bootstrap.ign" -o tsv`
echo $BOOTSTRAP_URL
export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "3.2.0" --arg url ${BOOTSTRAP_URL} '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 | tr -d '\n'`
echo $BOOTSTRAP_IGNITION

az deployment group create -g ${RESOURCE_GROUP} --template-file "${installdir}/04_bootstrap.json" --parameters bootstrapIgnition="${BOOTSTRAP_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters baseName="${INFRA_ID}" 

# Deploy masters

export MASTER_IGNITION=`cat ${installdir}/master.ign | base64 | tr -d '\n'`
echo $MASTER_IGNITION | base64 -d
az deployment group create -g ${RESOURCE_GROUP} --template-file "${installdir}/05_masters.json" --parameters masterIgnition="${MASTER_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" --parameters baseName="${INFRA_ID}" 

# wait for bootstrap to complete

openshift-install wait-for bootstrap-complete --dir=${installdir} --log-level info 

# remove boot strap resources

az network nsg rule delete -g ${RESOURCE_GROUP} --nsg-name ${INFRA_ID}-nsg --name bootstrap_ssh_in
az vm stop -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
az vm deallocate -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
az vm delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap --yes
az disk delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap_OSDisk --no-wait --yes
az network nic delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap-nic --no-wait
az storage blob delete --account-key ${ACCOUNT_KEY} --account-name ${CLUSTER_NAME}sa --container-name files --name bootstrap.ign
az network public-ip delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap-ssh-pip

# Deploy workers

export WORKER_IGNITION=`cat ${installdir}/worker.ign | base64 | tr -d '\n'`
echo $WORKER_IGNITION | base64 -d
az deployment group create -g ${RESOURCE_GROUP} --template-file "${installdir}/06_workers.json" --parameters workerIgnition="${WORKER_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters baseName="${INFRA_ID}" 

# sign certs for the worker nodes to join the cluster

oc get csr

oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

# update the dNS records

export PUBLIC_IP_ROUTER=`oc -n openshift-ingress get service router-default --no-headers | awk '{print $4}'`
echo $PUBLIC_IP_ROUTER
az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n '*.apps' -a ${PUBLIC_IP_ROUTER} --ttl 300

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${BASE_DOMAIN} -n '*.apps.${CLUSTER_NAME}' -a ${PUBLIC_IP_ROUTER} --ttl 300

az network private-dns record-set a create -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n '*.apps' --ttl 300

az network private-dns record-set a add-record -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n '*.apps' -a ${PUBLIC_IP_ROUTER}

# Get all the routes

oc get --all-namespaces -o jsonpath='{range .items[*]}{range .status.ingress[*]}{.host}{"\n"}{end}{end}' routes

./openshift-install --dir=${installdir} wait-for install-complete 


# To destroy cluster

openshift-install destroy cluster --dir ${installdir} --log-level=info

# enable etcd encryption

oc edit apiserver

# add this spec
spec:
  encryption:
    type: aescbc 

# verify encryption

oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

oc get authentication.operator.openshift.io -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'


# disable etcd encryption

oc edit apiserver

# add this spec
spec:
  encryption:
    type: identity 

# verify

oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

oc get authentication.operator.openshift.io -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

