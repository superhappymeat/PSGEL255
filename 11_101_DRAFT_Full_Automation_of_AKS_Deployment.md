![Global Enablement & Learning](https://gelgitlab.race.sas.com/GEL/utilities/writing-content-in-markdown/-/raw/master/img/gel_banner_logo_tech-partners.jpg)

**DRAFT**

<!--
```bash
#!/bin/bash -x

```
 -->

# Full automation of deployment

* [Create working folder](#create-working-folder)
* [Preparing tools](#preparing-tools)
  * [Getting the github projects cloned](#getting-the-github-projects-cloned)
  * [Building the docker images](#building-the-docker-images)
* [Authenticating to Azure and getting static credentials for Terraform](#authenticating-to-azure-and-getting-static-credentials-for-terraform)
* [Authenticate to Azure](#authenticate-to-azure)
  * [Figure the username of who just logged in](#figure-the-username-of-who-just-logged-in)
  * [Create an Azure Service Principal and store its info in a file](#create-an-azure-service-principal-and-store-its-info-in-a-file)
* [Prep the required files for the creation of the AKS cluster](#prep-the-required-files-for-the-creation-of-the-aks-cluster)
  * [Copy our public ssh key in the workspace](#copy-our-public-ssh-key-in-the-workspace)
  * [Create a .tfvars file for our first environment (gel01)](#create-a-tfvars-file-for-our-first-environment-gel01)
  * [Deploy new AKS cluster](#deploy-new-aks-cluster)
    * [Plan](#plan)
    * [show](#show)
    * [Apply](#apply)
    * [get the kubeconfig](#get-the-kubeconfig)
* [Prepare AKS and deploy Viya in it (gel01)](#prepare-aks-and-deploy-viya-in-it-gel01)
  * [Create vars.yaml for gel01](#create-varsyaml-for-gel01)
  * [Create directory structure](#create-directory-structure)
  * [Deploy Pre-reqs in AKS](#deploy-pre-reqs-in-aks)
  * [Deploy Viya and monitoring](#deploy-viya-and-monitoring)
* [Post-Deployment customizations](#post-deployment-customizations)
  * [Configure Ephemeral Storage](#configure-ephemeral-storage)
* [wrapping up](#wrapping-up)
  * [destroy everything](#destroy-everything)

## Create working folder

1. we will use a RACE machine as a jumphost to interact with AKS

    ```bash
    rm -rf ~/jumphost/
    mkdir -p ~/jumphost/
    mkdir -p ~/jumphost/workspace

    ```

1. We need to choose a default region

    ```bash
    AZUREREGION=${AZUREREGION:-eastus}

    echo ${AZUREREGION} > ~/jumphost/workspace/.azureregion.txt

    ```

## Preparing tools

### Getting the github projects cloned

1. Clone the IAC project

    ```bash
    cd  ~/jumphost/
    git clone https://github.com/sassoftware/viya4-iac-azure.git
    cd ~/jumphost/viya4-iac-azure
    git fetch --all
    git reset --hard 1.0.1
    #git checkout be3e9fac6e6a26bd9bdebdbcaebc7db3c3cb76f2

    #cd ~/jumphost/viya4-iac-azure ; git fetch --all ;  git reset --hard main"
    # If you want to try from main:
    #git reset --hard main
    #git pull

    ```

1. Clone the Deployment Project

    ```bash
    cd  ~/jumphost/
    git clone https://github.com/sassoftware/viya4-deployment.git
    cd ~/jumphost/viya4-deployment
    git fetch --all
    git reset --hard 1.0.1
    git reset --hard origin/main

    ```

### Building the docker images

1. Build the IAC container image

    ```bash
    ## build docker images
    cd  ~/jumphost/viya4-iac-azure
    time docker image build -t viya4-iac-azure .
    ## this should take about 1m20s

    ```

1. Build the deployment container image

    ```bash
    cd  ~/jumphost/viya4-deployment
    time docker image build -t viya4-deployment .
    ## this should take about 6m30s

    ```

1. Confirm that both image have been built

    ```bash
    docker image ls | grep viya4\-

    ```

1. Should return:

    ```log
    viya4-deployment  latest      176fce5e225a        10 seconds ago      1.1GB
    viya4-iac-azure   latest      0665c1cfc1a6        48 seconds ago      1.12GB
    ```

## Authenticating to Azure and getting static credentials for Terraform

## Authenticate to Azure

1. Do an AZ login first.

    ```bash
    if [ ! -d "$HOME/jumphost/workspace/.azure" ]; then
        echo "the Azure credentials directory is missing. Prompting you to authenticate"
        docker container  run -it --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            login
    fi
    ```

### Figure the username of who just logged in

1. Determine the userid of the person who just authenticated:

    ```bash
    docker container  run  --rm \
        -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
        --entrypoint az \
        viya4-iac-azure \
        ad signed-in-user show --query mailNickname \
            | sed  's|["\ ]||g' \
            | tee ~/jumphost/workspace/.student.txt

    docker container  run  --rm \
        -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
        --entrypoint az \
        viya4-iac-azure \
        ad signed-in-user show --query userPrincipalName \
            | sed  's|["\ ]||g' \
            | tee ~/jumphost/workspace/.email.txt

    ```

1. Confirm this user id is yours

    ```bash
    STUDENT=$(cat ~/jumphost/workspace/.student.txt)
    EMAIL=$(cat ~/jumphost/workspace/.email.txt)
    printf "\nThis should be your SAS Account:\n   ${STUDENT} (${EMAIL})  \nIf not, do not continue\n"

    if  [ "${STUDENT}" == "" ]; then
        echo "Can't figure out your student ID. Exiting"
        exit
    fi

    ```

### Create an Azure Service Principal and store its info in a file

1. Generate a Service Principal and store in a file

    ```bash

    STUDENT=$(cat ~/jumphost/workspace/.student.txt)

    TFCREDFILE=~/jumphost/workspace/.tf_creds

    if [ ! -f "$TFCREDFILE" ]; then
        touch ${TFCREDFILE}

        docker container  run  --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            ad sp \
            create-for-rbac --skip-assignment \
            --name http://${STUDENT} --query password \
            --output tsv \
            | grep -v "Found\ an\ existing" \
            | head -n 1 \
            | sed 's/^/TF_VAR_client_secret=/' \
            | tee -a ${TFCREDFILE}


        docker container run  --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            ad sp show --id http://${STUDENT} \
            --query appId --output tsv \
            | grep -v "Found\ an\ existing" \
            | head -n 1 \
            | sed 's/^/TF_VAR_client_id=/' \
            | tee -a ${TFCREDFILE}

        ## we need the APPID in an ENV var
        source <( grep TF_VAR_client_id ${TFCREDFILE}   )

        # set sas-gelsandbox as default subscription
        docker container  run -it --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            account set -s "sas-gelsandbox"

        ## assign the contributor role
        docker container  run -it --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            role assignment create \
            --assignee ${TF_VAR_client_id}  --role Contributor

        docker container  run  --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            account list --query "[?name=='sas-gelsandbox'].{TF_VAR_subscription_id:id}" \
            -o tsv \
            |  sed 's/^/TF_VAR_subscription_id=/' \
            | tee -a ${TFCREDFILE}

        docker container  run  --rm \
            -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
            --entrypoint az \
            viya4-iac-azure \
            account list --query "[?name=='sas-gelsandbox'].{TF_VAR_tenant_id:tenantId}" \
            -o tsv \
            |  sed 's/^/TF_VAR_tenant_id=/' \
            | tee -a ${TFCREDFILE}

    fi


    ```

## Prep the required files for the creation of the AKS cluster

### Copy our public ssh key in the workspace

1. just run this:

    ```bash
    cp  ~/.ssh/cloud-user_id_rsa.pub \
        ~/jumphost/workspace/id_rsa.pub
    cp  ~/.ssh/id_rsa \
        ~/jumphost/workspace/id_rsa

    ```

### Create a .tfvars file for our first environment (gel01)

```bash
# we add the race machine's IP to the authorized public access point
JUMPHOSTIP=$(curl https://ifconfig.me)

## look at how raphael does it
echo $JUMPHOSTIP

AZUREREGION=$(cat ~/jumphost/workspace/.azureregion.txt )
STUDENT=$(cat ~/jumphost/workspace/.student.txt)
EMAIL=$(cat ~/jumphost/workspace/.email.txt)


tee  ~/jumphost/workspace/gel01-vars.tfvars > /dev/null << EOF

prefix                               = "${STUDENT}-iac-viya4aks"
location                             = "${AZUREREGION}"
ssh_public_key                       = "/workspace/id_rsa.pub"

## General config
kubernetes_version                   = "1.18.10"

# no jump host machine
create_jump_public_ip                = false

# tags in azure
tags                                 = { "resourceowner" = "${EMAIL}" , project_name = "sasviya4", environment = "dev", gel_project = "deployviya4" }

## Azure Auth
# not required if already set in TF environment variables
#tenant_id                            = ${TENANT_ID}
#subscription_id                      = ${SUBSCRIPTION_ID}
#client_id                            = ${CLIENT_ID}
#client_secret                        = ${CLIENT_SECRET}

## Admin Access
# IP Ranges allowed to access all created cloud resources
default_public_access_cidrs         = ["109.232.56.224/27", "149.173.0.0/16", "194.206.69.176/28", "$JUMPHOSTIP/32"]

## Storage
# "dev" creates AzureFile, "standard" creates NFS server VM, "ha" creates Azure Netapp Files
storage_type                         = "dev"

default_nodepool_vm_type             = "Standard_D4_v2"
default_nodepool_min_nodes           = 1

node_pools_proximity_placement       = false

default_nodepool_availability_zones  = ["1"]
node_pools_availability_zone         = "1"


# AKS Node Pools config
node_pools = {
cas = {
    "machine_type" = "Standard_E4s_v3"         # # 4x32 w 64GBSSD @0.25
    "os_disk_size" = 200
    "min_nodes" = 1
    "max_nodes" = 8
    "node_taints" = ["workload.sas.com/class=cas:NoSchedule"]
    "node_labels" = {
    "workload.sas.com/class" = "cas"
    }
},
compute = {
    "machine_type" = "Standard_L8s_v2"        # # 8x64 w 80GBSSD 1.9TB NVMe @ 0.62
    "os_disk_size" = 200
    "min_nodes" = 1
    "max_nodes" = 1
    "node_taints" = ["workload.sas.com/class=compute:NoSchedule"]
    "node_labels" = {
    "workload.sas.com/class"        = "compute"
    "launcher.sas.com/prepullImage" = "sas-programming-environment"
    }
},
    connect = {
    "machine_type" = "Standard_L8s_v2"          # # 8x64 w 80GBSSD 1.9TB NVMe @ 0.62
    "os_disk_size" = 200
    "min_nodes" = 0
    "max_nodes" = 1
    "node_taints" = ["workload.sas.com/class=connect:NoSchedule"]
    "node_labels" = {
        "workload.sas.com/class"        = "connect"
        "launcher.sas.com/prepullImage" = "sas-programming-environment"
    }
    },
stateless = {
    "machine_type" = "Standard_D8s_v3"           # # 4x16 w 32GBSSD @ 0.19
    "os_disk_size" = 200
    "min_nodes" = 0
    "max_nodes" = 3
    "node_taints" = ["workload.sas.com/class=stateless:NoSchedule"]
    "node_labels" = {
    "workload.sas.com/class" = "stateless"
    }
},
stateful = {
    "machine_type" = "Standard_D8s_v3"          # # 4x16 w 32GBSSD @ 0.19
    "os_disk_size" = 200
    "min_nodes" = 0
    "max_nodes" = 3
    "node_taints" = ["workload.sas.com/class=stateful:NoSchedule"]
    "node_labels" = {
    "workload.sas.com/class" = "stateful"
    }
}
}

# Azure Postgres config
# # set this to "false" when using internal Crunchy Postgres and Azure Postgres is NOT needed
create_postgres                  = false

#postgres_sku_name                = "GP_Gen5_32"
#postgres_administrator_login     = "pgadmin"
#postgres_administrator_password  = "LNX_sas_123"
#postgres_ssl_enforcement_enabled = true

# Azure Container Registry
# We will use external orders and pull images from SAS Hosted registries
create_container_registry           = false
container_registry_sku              = "Standard"
container_registry_admin_enabled    = "false"
container_registry_geo_replica_locs = null
EOF


```

### Deploy new AKS cluster

#### Plan

```bash
# plan
docker container run --rm -it \
    --env-file $HOME/jumphost/workspace/.tf_creds \
    -v $HOME/jumphost/workspace:/workspace \
    viya4-iac-azure \
    plan \
    -var-file /workspace/gel01-vars.tfvars \
    -state /workspace/gel01-vars.tfstate \
    -out /workspace/gel01-vars.plan

```

#### show

```bash
docker container  run --rm -it \
    --env-file $HOME/jumphost/workspace/.tf_creds \
    -v $HOME/jumphost/workspace:/workspace \
    viya4-iac-azure \
    show \
     /workspace/gel01-vars.plan

```

#### Apply

```bash
time docker container  run --rm -it \
    --env-file $HOME/jumphost/workspace/.tf_creds \
    -v $HOME/jumphost/workspace:/workspace \
    viya4-iac-azure \
    apply \
    -auto-approve \
    -var-file /workspace/gel01-vars.tfvars \
    -state /workspace/gel01-vars.tfstate
## this should take between 2 and 16 minutes.
```

#### get the kubeconfig

with terraform

```bash
## get the kubeconfig file
docker container  run --rm \
    --env-file $HOME/jumphost/workspace/.tf_creds \
    -v $HOME/jumphost/workspace:/workspace \
    viya4-iac-azure \
    output \
    -state /workspace/gel01-vars.tfstate\
    kube_config > $HOME/jumphost/workspace/.kubeconfig_aks

mv ~/.kube/config ~/.kube/config_old
ln -s  $HOME/jumphost/workspace/.kubeconfig_aks  $HOME/.kube/config

```

or with az ( in case you lost your tfstate file)

```sh
rm -rf $HOME/jumphost/workspace/.kubeconfig_aks
touch $HOME/jumphost/workspace/.kubeconfig_aks
docker container  run  --rm \
    -v $HOME/jumphost/workspace/.azure/:/root/.azure/ \
    -v $HOME/jumphost/workspace/.kubeconfig_aks:/root/.kube/config \
    --entrypoint az \
    viya4-iac-azure \
    aks \
    get-credentials \
    --name canepg-iac-viya4aks-aks \
    --resource-group canepg-iac-viya4aks-rg

```

## Prepare AKS and deploy Viya in it (gel01)

### Create vars.yaml for gel01

from:

* <https://github.com/sassoftware/viya4-deployment/blob/main/docs/CONFIG-VARS.md>
* and
* <https://github.com/sassoftware/viya4-deployment/blob/main/examples/ansible-vars-azure.yaml>

```bash

AZUREREGION=$(cat ~/jumphost/workspace/.azureregion.txt )
STUDENT=$(cat ~/jumphost/workspace/.student.txt)
V4_CFG_INGRESS_FQDN=${STUDENT}vk.${AZUREREGION}.cloudapp.azure.com

tee  ~/jumphost/workspace/gel01-vars.yaml > /dev/null << EOF
## Cluster
PROVIDER: azure
CLUSTER_NAME: canepg-iac-viya4aks-aks
NAMESPACE: viya4

## MISC
DEPLOY: true # Set to false to stop at generating the manifest

#LOADBALANCER_SOURCE_RANGES: ['<cluster_nat_ip>/32']
LOADBALANCER_SOURCE_RANGES: ["109.232.56.224/27", "149.173.0.0/16", "194.206.69.176/28"]

## Storage
V4_CFG_MANAGE_STORAGE: true

## SAS API Access
V4_CFG_SAS_API_KEY: 'NUrshK9GqUIOEUAKZnShAnGRT2jpwUJt'
V4_CFG_SAS_API_SECRET: 'ATmBsOM5v9Wh20Fa'
V4_CFG_ORDER_NUMBER: 9CDZDD

## CR Access
# V4_CFG_CR_USER: <container_registry_user>
# V4_CFG_CR_PASSWORD: <container_registry_password>

## Ingress
V4_CFG_INGRESS_TYPE: ingress
V4_CFG_INGRESS_FQDN: "${V4_CFG_INGRESS_FQDN}"
V4_CFG_TLS_MODE: "full-stack" # [full-stack|front-door|disabled]

## Postgres
V4_CFG_POSTGRES_TYPE: internal
V4_CFG_POSTGRES_ADMIN_LOGIN: <existing_pg_user>
V4_CFG_POSTGRES_PASSWORD: <existing_pg_password>
V4_CFG_POSTGRES_FQDN: <existing_pg_fqdn>
V4_CFG_POSTGRES_PORT: 5432

## LDAP
V4_CFG_EMBEDDED_LDAP_ENABLE: true

## Consul UI
V4_CFG_CONSUL_ENABLE_LOADBALANCER: false

## SAS/CONNECT
V4_CFG_CONNECT_ENABLE_LOADBALANCER: true

V4_CFG_CADENCE_NAME: 'lts'
V4_CFG_CADENCE_VERSION: '2020.1'



V4_CFG_CAS_WORKER_COUNT: '3'
V4_CFG_CAS_ENABLE_BACKUP_CONTROLLER: true
V4_CFG_CAS_ENABLE_LOADBALANCER: true

EOF


```

### Create directory structure

```bash
mkdir -p $HOME/jumphost/workspace/deploy/
mkdir -p $HOME/jumphost/workspace/deploy/canepg-iac-viya4aks-aks
mkdir -p $HOME/jumphost/workspace/deploy/canepg-iac-viya4aks-aks/viya4
mkdir -p $HOME/jumphost/workspace/deploy/canepg-iac-viya4aks-aks/viya4/site-config

```

### Deploy Pre-reqs in AKS

```bash
docker container run -it \
  -v $HOME/jumphost/workspace/deploy:/data \
  -v $HOME/jumphost/workspace/.kubeconfig_aks:/config/kubeconfig \
  -v $HOME/jumphost/workspace/gel01-vars.yaml:/config/config \
  -v $HOME/jumphost/workspace/gel01-vars.tfstate:/config/tfstate \
  -v $HOME/jumphost/workspace/id_rsa:/config/jump_svr_private_key \
  viya4-deployment \
    --tags "baseline,install"

```

### Deploy Viya and monitoring

1. step

    ```sh
    docker container run -it \
        -v $HOME/jumphost/workspace/deploy:/data \
        -v $HOME/jumphost/workspace/.kubeconfig_aks:/config/kubeconfig \
        -v $HOME/jumphost/workspace/gel01-vars.yaml:/config/config \
        -v $HOME/jumphost/workspace/gel01-vars.tfstate:/config/tfstate \
        -v $HOME/jumphost/workspace/id_rsa:/config/jump_svr_private_key \
        viya4-deployment \
        --tags "install,baseline,viya,cluster-logging,cluster-monitoring,viya-monitoring"

    ```

## Post-Deployment customizations

### Configure Ephemeral Storage

```bash
cd $HOME/jumphost/
git clone https://gelgitlab.race.sas.com/GEL/deployment/kev
cd $HOME/jumphost/kev/nodestrap
kustomize build | kubectl apply -f -

```

## wrapping up

### destroy everything

```sh
docker container  run -it --rm \
    --env-file $HOME/jumphost/workspace/.tf_creds \
    -v $HOME/jumphost/workspace:/workspace \
    viya4-iac-azure \
    destroy \
    -var-file /workspace/gel01-vars.tfvars \
    -state /workspace/gel01-vars.tfstate \
    -lock=false \
    -force \
    -input=false

```
