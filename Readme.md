# Create install config
openshift-install create install-config --dir=azureocp

export CLUSTER_NAME=ocpupi
export AZURE_REGION=australiasoutheast
export SSH_KEY=`cat ~/.ssh/id_rsa`
export BASE_DOMAIN=themallams.com
export BASE_DOMAIN_RESOURCE_GROUP=aks-resource-group

export KUBECONFIG=azureocp/auth/kubeconfig   

# Create Kube manifest files
openshift-install create manifests --dir=azureocp    

rm -rf azureocp/openshift/99_openshift-cluster-api_master-machines-*.yaml 
rm -rf azureocp/openshift/99_openshift-cluster-api_worker-machineset-*.yaml


export INFRA_ID=`grep infrastructureName  azureocp/manifests/cluster-infrastructure-02-config.yml | awk -F ':' '{print $2}'`

export RESOURCE_GROUP=`grep resourceGroupName  azureocp/manifests/cluster-infrastructure-02-config.yml | awk -F ':' '{print $2}'`

# Create ignition files
openshift-install create ignition-configs --dir azureocp 

# create Resource Group in azure to hold the resources
az group create --name ${RESOURCE_GROUP} --location ${AZURE_REGION}

# create identity 

az identity create -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity

export PRINCIPAL_ID=`az identity show -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity --query principalId --out tsv`

export RESOURCE_GROUP_ID=`az group show -g ${RESOURCE_GROUP} --query id --out tsv`

az role assignment create --assignee "${PRINCIPAL_ID}" --role 'Contributor' --scope "${RESOURCE_GROUP_ID}"

# Create an Azure storage account to store the VHD cluster image
az storage account create -g ${RESOURCE_GROUP} --location ${AZURE_REGION} --name ${CLUSTER_NAME}sa --kind Storage --sku Standard_LRS

# get account key from storage account

export ACCOUNT_KEY=`az storage account keys list -g ${RESOURCE_GROUP} --account-name ${CLUSTER_NAME}sa --query "[0].value" -o tsv`

export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.8/data/data/rhcos.json | jq -r .azure.url`

az storage container create --name vhd --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY}

az storage blob copy start --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "${VHD_URL}"

tstatus="unknown"
while [ "$tstatus" != "success" ]
do
  tstatus=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -o tsv --query properties.copy.status`
  echo $tstatus
done

az storage container create --name files --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} --public-access blob

az storage blob upload --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c "files" -f "azureocp/bootstrap.ign" -n "bootstrap.ign"

 # Create public and private dns zones

az network dns zone create -g ${BASE_DOMAIN_RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}

az network private-dns zone create -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}


# Create VNET

az deployment group create -g ${RESOURCE_GROUP} --template-file "azureocp/01_vnet.json" --parameters baseName="${INFRA_ID}"

az network private-dns link vnet create -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n ${INFRA_ID}-network-link -v "${INFRA_ID}-vnet" -e false

# Creae deployment group

export VHD_BLOB_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c vhd -n "rhcos.vhd" -o tsv`

az deployment group create -g ${RESOURCE_GROUP} --template-file "azureocp/02_storage.json" --parameters vhdBlobURL="${VHD_BLOB_URL}" --parameters baseName="${INFRA_ID}" 

# create networking and loadbalancing components
az deployment group create -g ${RESOURCE_GROUP} --template-file "azureocp/03_infra.json" --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" --parameters baseName="${INFRA_ID}" 


export PUBLIC_IP=`az network public-ip list -g ${RESOURCE_GROUP} --query "[?name=='${INFRA_ID}-master-pip'] | [0].ipAddress" -o tsv`

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n api -a ${PUBLIC_IP} --ttl 60

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${BASE_DOMAIN} -n api.${CLUSTER_NAME} -a ${PUBLIC_IP} --ttl 60

export BOOTSTRAP_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c "files" -n "bootstrap.ign" -o tsv`
export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "3.2.0" --arg url ${BOOTSTRAP_URL} '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 | tr -d '\n'`


az deployment group create -g ${RESOURCE_GROUP} --template-file "azureocp/04_bootstrap.json" --parameters bootstrapIgnition="${BOOTSTRAP_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters baseName="${INFRA_ID}" 


export MASTER_IGNITION=`cat azureocp/master.ign | base64 | tr -d '\n'`

az deployment group create -g ${RESOURCE_GROUP} --template-file "azureocp/05_masters.json" --parameters masterIgnition="${MASTER_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" --parameters baseName="${INFRA_ID}" 

# wait for bootstrap to complete

./openshift-install wait-for bootstrap-complete --dir=azureocp --log-level info 

# remove boot strap resources

az network nsg rule delete -g ${RESOURCE_GROUP} --nsg-name ${INFRA_ID}-nsg --name bootstrap_ssh_in
az vm stop -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
az vm deallocate -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
az vm delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap --yes
az disk delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap_OSDisk --no-wait --yes
az network nic delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap-nic --no-wait
az storage blob delete --account-key ${ACCOUNT_KEY} --account-name ${CLUSTER_NAME}sa --container-name files --name bootstrap.ign
az network public-ip delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap-ssh-pip

# workers

export WORKER_IGNITION=`cat azureocp/worker.ign | base64 | tr -d '\n'`

az deployment group create -g ${RESOURCE_GROUP} --template-file "azureocp/06_workers.json" --parameters workerIgnition="${WORKER_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters baseName="${INFRA_ID}" 

# sign certs for the worker nodes to join the cluster

oc get csr

oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve

oc -n openshift-ingress get service router-default

export PUBLIC_IP_ROUTER=`oc -n openshift-ingress get service router-default --no-headers | awk '{print $4}'`

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n *.apps -a ${PUBLIC_IP_ROUTER} --ttl 300

az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${BASE_DOMAIN} -n *.apps.${CLUSTER_NAME} -a ${PUBLIC_IP_ROUTER} --ttl 300

az network private-dns record-set a create -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n *.apps --ttl 300

az network private-dns record-set a add-record -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n *.apps -a ${PUBLIC_IP_ROUTER}

oc get --all-namespaces -o jsonpath='{range .items[*]}{range .status.ingress[*]}{.host}{"\n"}{end}{end}' routes

./openshift-install --dir=azureocp wait-for install-complete 


# To destroy cluster

openshift-install destroy cluster --dir azureocp --log-level=info