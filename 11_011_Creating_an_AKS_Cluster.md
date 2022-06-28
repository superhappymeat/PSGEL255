![Global Enablement & Learning](https://gelgitlab.race.sas.com/GEL/utilities/writing-content-in-markdown/-/raw/master/img/gel_banner_logo_tech-partners.jpg)

# Creating an AKS cluster

* [Projects](#projects)
  * [GitHub Project: SAS Viya 4 IaC for Azure](#github-project-sas-viya-4-iac-for-azure)
* [Initialize your working environment (please READ)](#initialize-your-working-environment-please-read)
  * [Request your gel-sandbox access](#request-your-gel-sandbox-access)
  * [Open the Azure Cloud Shell](#open-the-azure-cloud-shell)
  * [Get the payload tarball in Cloud Shell](#get-the-payload-tarball-in-cloud-shell)
  * [Install Azure CLI](#install-azure-cli)
  * [Authenticate to Azure with the CLI](#authenticate-to-azure-with-the-cli)
  * [Set the Azure CLI defaults](#set-the-azure-cli-defaults)
  * [Student Differentiation](#student-differentiation)
* [Install and Configure Terraform](#install-and-configure-terraform)
  * [Set up Terraform](#set-up-terraform)
  * [Obtain the Terraform templates](#obtain-the-terraform-templates)
  * [Figure out some Azure credentials for Terraform](#figure-out-some-azure-credentials-for-terraform)
  * [Initialize Terraform](#initialize-terraform)
* [Provision the AKS cluster with Terraform](#provision-the-aks-cluster-with-terraform)
  * [Create an ssh key](#create-an-ssh-key)
  * [Get the email for the Azure tags](#get-the-email-for-the-azure-tags)
  * [Customize the Terraform variables](#customize-the-terraform-variables)
  * [Generate a Terraform plan](#generate-a-terraform-plan)
* [Stand up AKS cluster and Validate](#stand-up-aks-cluster-and-validate)
  * [Apply the Terraform plan](#apply-the-terraform-plan)
  * [In case it did not work for you](#in-case-it-did-not-work-for-you)
  * [Getting the kubectl config file for your newly built AKS cluster](#getting-the-kubectl-config-file-for-your-newly-built-aks-cluster)
  * [Disable the authorized IP range for the Kubernetes API](#disable-the-authorized-ip-range-for-the-kubernetes-api)
  * [Configure kubectl auto-completion](#configure-kubectl-auto-completion)
* [IMPORTANT FOR THE NEXT TIME : DO NOT REDO EVERYTHING](#important-for-the-next-time--do-not-redo-everything)

## Projects

### GitHub Project: SAS Viya 4 IaC for Azure

There is a called "SAS Viya 4 IaC for Azure" that uses Terraform and Modules from Terraform Registry to create and configure Azure infrastructure components needed to deploy SAS Viya 4.x products on AKS.

<https://github.com/sassoftware/viya4-iac-azure>

We will get and use the Terraform templates from there to automate the AKS cluster provisioning.

## Initialize your working environment (please READ)

In the whole AKS hands-on we will use the **Azure Cloud Shell**.

Azure Cloud Shell is an interactive, authenticated, browser-accessible shell for managing Azure resources. It provides the flexibility of the shell experience (Bash or PowerShell) directly from your web browser.

Behind the scene Azure starts a small container for that with everything you need pre-installed (ansible, az CLI, terraform, git, etc...), so there is no longer a need to provision a jumphost machine to operate your deployment in Azure, just start an Azure Shell and launch the deployment of Viya 4 in AKS from there.

If it is the first time you are using the Azure Cloud Shell, you will be asked to attach an azure file share to persist the files.
If you are not prompted with the question, no worries it means that there is already a storage associated to your Cloud Shell session, and you can proceeed with the Hands-on.

_Useful tip : unfortunately in the Azure shell, the left and right-clicks to copy-paste (that is provided with modern linux terminal client such as MobaXterm) is NOT available. However you can use either the contextual menus , or the "CTRL+C" and "SHIFT+INSERT" keys to copy paste (Thanks @senad.jukic for reporting this feature)._

### Request your gel-sandbox access

* Send this [email](mailto:dlmanager@wnt.sas.com?subject=SUBSCRIBE%20azure_gelsandbox) without any change.

* If the link above does not open your default email program, see [this](https://duckduckgo.com/?q=mailto+protocol+default) or simply email dlmanager@wnt.sas.com with subject "SUBSCRIBE azure_gelsandbox".

* DLManager will send an email response in a few minutes confirming your membership in the GELSANDBOX Azure subscription.

  ![DL Manager email](img/2020-09-10-10-02-25.png)

* As said in the email you will have to wait for about **15 minutes** (for the AD replication to propagate the change) before continuing with the prerequisite steps.

### Open the Azure Cloud Shell

Open the Azure Portal to ensure you have been added in the "sas-gelsandbox" subscription and can see the resource groups.

* Open <https://portal.azure.com/#home>

  ![cloud portal](img/2020-11-19-08-41-11.png)

If you see "sas-gelsandbox" in the list of subscription, you can follow the next steps.

* Click on Cloud-shell icon

  ![cloudshell](img/2020-07-21-17-13-36.png)

Note : you can also directly open the Cloud Shell from this URL : <https://shell.azure.com/>

* If you see the screen below, click on "Bash"

  ![bashorps](img/2020-09-07-17-50-49.png)

* If it is the first time you are using the Azure Cloud Shell, you are asked to attach an azure file share to persist the files you will create during the Cloud Shell session.

  ![storage](img/2020-07-21-17-16-32.png)

* If you are not prompted with the question, no worries. It just means that there is already a storage account associated to your Cloud Shell sessions. The Cloud Shell terminal window will open up and you can proceeed with the next [instructions](#get-the-payload-tarball-in-cloud-shell).

* **Otherwhise** click on "Show Advanced Settings", so you can select the resource group, the storage account and a name for the File share that will be attached to the Cloud Shell sessions.

    ![create storage](img/2020-07-23-10-08-52.png)

* Fulfill the form as in the screenshot (Use your own subscription or create a new Resource Group using your own SAS ID in the name so it will be easier to find it).

* Click on "Create Storage" and after a little while, you should see a nice new bash window.

    ![prompt](img/2020-07-21-17-29-17.png)

### Get the payload tarball in Cloud Shell

There are a few things that we need to perform the Viya deployment in AKS (it is discussed in the [Prereqs](11_012_Performing_Prereqs_in_AKS.md) section).

The "payload" archive includes all we need and is automatically generated and made available on our good old gelweb server : <https://gelweb.race.sas.com/scripts/PSGEL255/payload/>

Since we will need the content of the payload archive for the next steps and we are preparing the Cloud Shell environment, let's upload it in our Cloud Shell environment.

Click on the "Upload/Download files" button at the top.

![upload](img/2020-07-21-17-41-17.png)

Click on "Upload" and copy-paste this URL : <https://gelweb.race.sas.com/scripts/PSGEL255/payload/payload.tgz>

You will see a new dialog that confirm that the archive has been uploaded.

![payload](img/2020-08-26-11-33-27.png)

The .tgz file has been uploaded in your home directory (/home/raphael in my case).

* Confirm it by running:

  ```sh
  ls -al ~
  ```

* Now, let's move and extract it to a known location.

  ```sh
  rm -Rf ~/clouddrive/payload
  mv ~/payload*.tgz ~/clouddrive/payload.tgz
  cd ~/clouddrive
  tar -xvf payload.tgz && rm payload.tgz
  ```

* Check that you have everything in the payload folder:

  ```sh
  ls -l ~/clouddrive/payload
  ```

* you should see something like :

    ```log
    erwan@Azure:~/clouddrive$ ls -l ~/clouddrive/payload
    total 0
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 cheatcodes
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 gelldap
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 gel_OKViya4
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 kustomize
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 mirrormgr
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 nginx
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 ops4viya
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 orders
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 viya4-monitoring
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 viya4ordercli
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 viyaarkcd
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 viya_manifests
    drwxrwxrwx 2 erwan erwan 0 Nov 18 18:20 workshop
    ```

### Install Azure CLI

* Nothing to do since the "az" cli is pre-installed in the Azure Cloud Shell :)

    ```bash
    #show az CLI version
    az version
    ```

### Authenticate to Azure with the CLI

Cloud Shell is automatically authenticated under the initial account signed-in with.

### Set the Azure CLI defaults

* We need to define the location we want. Let's list the locations, and choose one (eastus):

    ```bash
    az account list-locations -o table
    az configure --defaults location=eastus

    # The default region we choose is "eastus" but you can choose something else if you want:
    AZUREREGION=${AZUREREGION:-eastus}

    echo ${AZUREREGION} > ~/azureregion.txt
    cat ~/azureregion.txt
    az configure --defaults location=${AZUREREGION}
    # Save the AZUREREGION info for next time we re-login
    ansible localhost -m lineinfile \
        -a "dest=~/.bashrc \
            regexp='^export AZUREREGION' \
            line='export AZUREREGION=$(cat ~/azureregion.txt)'" \
            --diff

    ```

  <details><summary>Click here to see a sample output of the available regions</summary>

  ```log
  [cloud-user@pdcesx04212 ~]$     az account list-locations -o table
  DisplayName           Latitude    Longitude    Name
  --------------------  ----------  -----------  ------------------
  East Asia             22.267      114.188      eastasia
  Southeast Asia        1.283       103.833      southeastasia
  Central US            41.5908     -93.6208     centralus
  East US               37.3719     -79.8164     eastus
  East US 2             36.6681     -78.3889     eastus2
  West US               37.783      -122.417     westus
  North Central US      41.8819     -87.6278     northcentralus
  South Central US      29.4167     -98.5        southcentralus
  North Europe          53.3478     -6.2597      northeurope
  West Europe           52.3667     4.9          westeurope
  Japan West            34.6939     135.5022     japanwest
  Japan East            35.68       139.77       japaneast
  Brazil South          -23.55      -46.633      brazilsouth
  Australia East        -33.86      151.2094     australiaeast
  Australia Southeast   -37.8136    144.9631     australiasoutheast
  South India           12.9822     80.1636      southindia
  Central India         18.5822     73.9197      centralindia
  West India            19.088      72.868       westindia
  Canada Central        43.653      -79.383      canadacentral
  Canada East           46.817      -71.217      canadaeast
  UK South              50.941      -0.799       uksouth
  UK West               53.427      -3.084       ukwest
  West Central US       40.890      -110.234     westcentralus
  West US 2             47.233      -119.852     westus2
  Korea Central         37.5665     126.9780     koreacentral
  Korea South           35.1796     129.0756     koreasouth
  France Central        46.3772     2.3730       francecentral
  France South          43.8345     2.1972       francesouth
  Australia Central     -35.3075    149.1244     australiacentral
  Australia Central 2   -35.3075    149.1244     australiacentral2
  UAE Central           24.466667   54.366669    uaecentral
  UAE North             25.266666   55.316666    uaenorth
  South Africa North    -25.731340  28.218370    southafricanorth
  South Africa West     -34.075691  18.843266    southafricawest
  Switzerland North     47.451542   8.564572     switzerlandnorth
  Switzerland West      46.204391   6.143158     switzerlandwest
  Germany North         53.073635   8.806422     germanynorth
  Germany West Central  50.110924   8.682127     germanywestcentral
  Norway West           58.969975   5.733107     norwaywest
  Norway East           59.913868   10.752245    norwayeast
  [cloud-user@pdcesx04212 ~]$ az configure --defaults location=eastus
  [cloud-user@pdcesx04212 ~]$
  ```

  </details>

<!--
  ```sh
  #set default AZ region
  az configure --defaults location=eastus
  ```
-->

* Now, let's list the subscriptions we have access to and choose the default one:

  ```sh
  az account list -o table
  ```

  if sas-gelsandbox is not the default, then set it :

  ```bash
  az account set -s "sas-gelsandbox"
  ```

  <details><summary>Click here to see the expected output</summary>

  ```log
  [cloud-user@pdcesx04212 ~]$ az account set -s "sas-gelsandbox"
  [cloud-user@pdcesx04212 ~]$ az account list -o table
  Name            CloudName    SubscriptionId                        State    IsDefault
  --------------  -----------  ------------------------------------  -------  -----------
  sas-gelsandbox  AzureCloud   c973059c-87f4-4d89-8724-a0da5fe4ad5c  Enabled  True
  Azure for PSD   AzureCloud   252972f2-a343-49b6-96d7-e619e918542a  Enabled  False
  GLE             AzureCloud   b91ae007-b39e-488f-bbbf-bc504d0a8917  Enabled  False
  Sandbox         AzureCloud   5509fbdf-fcde-4e29-be52-558b41db7221  Enabled  False
  [cloud-user@pdcesx04212 ~]$
  ```

</details>

<!--
  ```sh
  #set default AZ account
  az account set -s "sas-gelsandbox"
  ```
  -->

### Student Differentiation

This step is important and **mandatory** as it will ensure that what you create in Azure will have your SAS ID in its name. It will prevents conflicts and ease the tracking of the resources and costs.

* Run the command below to add your SAS user ID in the STUDENT variable:

  ```bash
  # add your SAS user ID in the STUDENT variable
  STUDENT=$(az ad signed-in-user show --query mailNickname | sed -e 's/^"//' -e 's/"$//')
  #STUDENT=$(az ad signed-in-user show --query displayName | sed -e 's/^"//' -e 's/"$//' | tr -d ' ')
  echo $STUDENT > ~/student.txt
  cat ~/student.txt
  ```

* Save the STUDENT info for next time we re-login

  ```bash
  # Save the STUDENT info for next time we re-login
  ansible localhost -m lineinfile \
    -a "dest=~/.bashrc \
        regexp='^export STUDENT' \
        line='export STUDENT=$(cat ~/student.txt)'" \
        --diff
  ```

_Note: the ansible command results in a message saying “CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in a future release.” And then a separate Ansible warning about no inventory have been parsed. This is fine – we use an ansible version based on Python 2 and we ran the command on localhost, neither of these messages are anything to worry about._

## Install and Configure Terraform

### Set up Terraform

* Nothing to do since terraform is pre-installed in the Azure Cloud Shell :)

  ```bash
  #show default terraform version
  terraform version
  ```

* ...But... actually we want to pin the Cloud Shell Terraform to the version required by the "SAS Viya 4 IaC for Azure" tool.

    ```bash
    # we stick to 13.4 bc it is the required version from https://github.com/sassoftware/viya4-iac-azure
    TFVERS=0.13.4
    echo "[INFO] installing terraform $TFVERS..."
    mkdir -p ~/bin
    cd ~/bin
    rm -Rf ~/bin/terraform
    curl -o terraform.zip -s https://releases.hashicorp.com/terraform/${TFVERS}/terraform_${TFVERS}_linux_amd64.zip
    unzip terraform.zip && rm -f terraform.zip
    export PATH=$PATH:$HOME/bin
    $HOME/bin/terraform version
    ```

* If you see a message "Your version of Terraform is out of date!" you can safely ignore it.

* Why ?

  Azure keeps updating the Terraform version and we want to be sure that we use a supported version for our [Viya IaC templates](https://github.com/sassoftware/viya4-iac-azure).
  For example when the available version in Azure Shell was v0.1.3.1 it did not play well with our TF templates :( (see <https://github.com/hashicorp/terraform/issues/26011>).
  At the moment TF v.0.13.* is in lot of flux - <https://github.com/hashicorp/terraform/pulse>

### Obtain the Terraform templates

The Terraform templates that we need comes from this [project](https://github.com/sassoftware/viya4-iac-azure).
Since they are public, they are not included in the payload tarball.

* Get the Terraform artifacts from github and checkout a specific version

    ```bash
    rm -Rf ~/clouddrive/project/aks/viya4-iac-azure
    mkdir -p ~/clouddrive/project/aks/
    cd ~/clouddrive/project/aks/
    git clone https://github.com/sassoftware/viya4-iac-azure.git

    # Instead of being at the mercy of the latest changes, we pin to a specific version
    cd ~/clouddrive/project/aks/viya4-iac-azure/
    git fetch --all
    IAC_AZURE_TAG=1.0.1

    git checkout tags/${IAC_AZURE_TAG}

    ```

### Figure out some Azure credentials for Terraform

* Create an Azure service principal

  We need to create and grant an azure Service Principal.
  See this [page](https://gitlab.sas.com/CloudOps4Viya/k8s-quickstarts-tf/azure-aks-4-viya/-/blob/master/docs/user/AzureHelpTopics.md) for more details.
  A Service Principal is effectively a "user" that you create to use with automated tools, like Terraform, to access Azure services on your behalf. You give it a role with only the permissions needed to execute the tasks that the Service Principal performs on your behalf.

  If you have sent us an email to request your sas-gelsandbox access (in this [step](#request-your-gel-sandbox-access)), we  have added you in the group that gives you the required permissions to do so.

  Note that if you are going through these instructions **for the second time**, you can skip this step (because in the following insructions we ask you to  stored the credential information in the TF_CLIENT_CREDS file that is keep in your storage account associated to your Azure Shell sesssion).

  ```sh
  # Create an Azure Service Principal and get associated Password and Id
  # You only have to do it once ! if the credential file already we skip
  TFCREDFILE=~/clouddrive/TF_CLIENT_CREDS
  if [ ! -f "$TFCREDFILE" ]; then
    SP_PASSWD=$(az ad sp create-for-rbac --skip-assignment --name http://${STUDENT} --query password --output tsv)
    SP_APPID=$(az ad sp show --id http://${STUDENT} --query appId --output tsv)
    # give the "Contributor" role to your Azure SP
    # You only have to do it once !
    az role assignment create --assignee $SP_APPID --role Contributor
  fi
  ```

  ```sh
  # export the required values in TF required environment variables
  export TF_VAR_subscription_id=$(az account list --query "[?name=='sas-gelsandbox'].{id:id}" -o tsv)
  export TF_VAR_tenant_id=$(az account list --query "[?name=='sas-gelsandbox'].{tenantId:tenantId}" -o tsv)
  export TF_VAR_client_id=${SP_APPID}
  export TF_VAR_client_secret=${SP_PASSWD}

  printf "TF_VAR_subscription_id   -->   ${TF_VAR_subscription_id}
  TF_VAR_tenant_id         -->   ${TF_VAR_tenant_id}
  TF_VAR_client_id      -->   ${TF_VAR_client_id}
  TF_VAR_client_secret  -->   ${TF_VAR_client_secret}\n"
  ```

* Make sure that IDs have been assigned to each of the four TF_xxx environment variables.
* Let's persist the required Terraform IDs, so we don't have to re-instantiate them each time we use the TF plan.

  ```sh
  cd ~/clouddrive
  # save the TF environment variables value for th next time
  tee TF_CLIENT_CREDS > /dev/null << EOF
  export TF_VAR_subscription_id=${TF_VAR_subscription_id}
  export TF_VAR_tenant_id=${TF_VAR_tenant_id}
  export TF_VAR_client_id=${TF_VAR_client_id}
  export TF_VAR_client_secret=${TF_VAR_client_secret}
  EOF
  chmod u+x TF_CLIENT_CREDS
  . ./TF_CLIENT_CREDS
  # Force TF_CLIENT_CREDS to run next time we re-login
  ansible localhost -m lineinfile -a "dest=~/.bashrc line='source $HOME/clouddrive/TF_CLIENT_CREDS'" --diff
  ```

<!-- cheat code
  ```bash
  # Initialize or get TF Credentials

  TFCREDFILE=~/clouddrive/TF_CLIENT_CREDS
  if [ -f "$TFCREDFILE" ]; then
      echo "Let's use the existing TF Credential file"
      cd ~/clouddrive
      . ./TF_CLIENT_CREDS
  else
      #get required TF Ids from Azure
      echo "No existing file...Get and Set SPN and variables for TF CREDs"
      TF_VAR_subscription_id=$(az account list --query "[?name=='sas-gelsandbox'].{id:id}" -o tsv)
      TF_VAR_tenant_id=$(az account list --query "[?name=='sas-gelsandbox'].{tenantId:tenantId}" -o tsv)
      TF_VAR_client_secret=$(az ad sp create-for-rbac --skip-assignment --name http://${STUDENT} --query password --output tsv)
      TF_VAR_client_id=$(az ad sp show --id http://${STUDENT} --query appId --output tsv)

      # ensure the SP has the Contributor role
      az role assignment create --assignee $SP_APPID --role Contributor

      #Persist Azure Service Principal login and secret, so we don't have to re-instanciate them each time we use the TF plan.
      echo "export TF_VAR_subscription_id=${TF_VAR_subscription_id}" > $TFCREDFILE
      echo "export TF_VAR_tenant_id=${TF_VAR_tenant_id}" >> $TFCREDFILE
      echo "export TF_VAR_client_id=${TF_VAR_client_id}" >> $TFCREDFILE
      echo "export TF_VAR_client_secret=${TF_VAR_client_secret}" >> $TFCREDFILE
      chmod u+x $TFCREDFILE
      cd ~/clouddrive
      . ./TF_CLIENT_CREDS
       ansible localhost -m lineinfile -a "dest=~/.bashrc line='source $HOME/clouddrive/TF_CLIENT_CREDS'" --diff
  fi
  ```
-->

### Initialize Terraform

* Run this command to initialize terraform in our environment.

  ```bash
  cd ~/clouddrive/project/aks/viya4-iac-azure
  #terraform init
  $HOME/bin/terraform init
  ```

* the end of the output should look like:

  ![tf init](img/2020-11-19-10-09-32.png)

  The important part is **"Terraform has been successfully initialized!"**

## Provision the AKS cluster with Terraform

### Create an ssh key

* An ssh key needs to be created in case we create a jumphost VM with Terraform

    ```bash
    # ensure there is a .ssh dir in $HOME
    ansible localhost -m file \
        -a "path=$HOME/.ssh mode=0700 state=directory"


    # ensure there is an ssh key that we can use
    ansible localhost -m openssh_keypair \
        -a "path=~/.ssh/id_rsa type=rsa size=2048" --diff
    ```

### Get the email for the Azure tags

* Get the student's email as we will use it to tag our Azure resources

    ```bash
    # get azure user email address
    az ad signed-in-user show --query userPrincipalName \
            | sed  's|["\ ]||g' \
            | tee ~/.email.txt
    # store email address in a variable to use later in TF variable
    EMAIL=$(cat ~/.email.txt)
    ```

### Customize the Terraform variables

There are many knobs that you can play with in the TF templates using the variables.
See this [page](https://github.com/sassoftware/viya4-iac-azure/blob/main/variables.tf) in the IAC-Azure project for the list of available variables and default values.

You can also consult [this page](https://github.com/sassoftware/viya4-iac-azure/blob/main/docs/CONFIG-VARS.md) for a complete and readable list.

As an example, try to find which variable to use to increase the sizing of my Azure Database for PostgreSQL.

Let's create our Terraform variables file:

* ```gel-vars.tfvars``` with multiple node pools specialized : cas node pool, compute node pool, stateless and stateful node pool.
<!-- * ```gel-vars-minimal.tfvars``` minimal version with a default node pool containing only one node (if more resources are needed we rely on the AKS autoscaling feature) -->

* Run this block of commands to populate the TF variables file:

  ```bash
  # Populate the TF variables file
  cd ~/clouddrive/project/aks/viya4-iac-azure
  # we add the CloudShell IP to the authorized public access point
  #CLOUDSHELLIP=$(curl https://ifconfig.me)
  CLOUDSHELLIP=$(dig +short myip.opendns.com @resolver1.opendns.com)
  # todo cheatcode to test IP adres with if [[ $ip =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then stat='good'; else stat='bad'
  echo $CLOUDSHELLIP

  AZUREREGION=$(cat ~/azureregion.txt)

  tee  ~/clouddrive/project/aks/viya4-iac-azure/gel-vars.tfvars > /dev/null << EOF

  prefix                               = "${STUDENT}viya4aks"
  location                             = "${AZUREREGION}"
  ssh_public_key                       = "~/.ssh/id_rsa.pub"

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
  default_public_access_cidrs         = ["109.232.56.224/27", "149.173.0.0/16", "194.206.69.176/28", "$CLOUDSHELLIP/32"]

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
      "machine_type" = "Standard_D8s_v3"           # # 8x32 w 64GBSSD @ 0.38
      "os_disk_size" = 200
      "min_nodes" = 0
      "max_nodes" = 3
      "node_taints" = ["workload.sas.com/class=stateless:NoSchedule"]
      "node_labels" = {
        "workload.sas.com/class" = "stateless"
      }
    },
    stateful = {
      "machine_type" = "Standard_D8s_v3"          # # 8x32 w 64GBSSD @ 0.38
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
  create_postgres                  = true

  postgres_sku_name                = "GP_Gen5_8"
  postgres_administrator_login     = "pgadmin"
  postgres_administrator_password  = "LNX_sas_123"
  postgres_ssl_enforcement_enabled = true

  # Azure Container Registry
  # We will use external orders and pull images from SAS Hosted registries
  create_container_registry           = false
  container_registry_sku              = "Standard"
  container_registry_admin_enabled    = "false"
  container_registry_geo_replica_locs = null
  EOF


  ```

* Take a moment to review and understand the content of the files. It defines how our Kubernetes cluster will look like in Azure.
* To easily edit the variable file, issue the following command in your Cloud Shell:

  ```sh
  code ~/clouddrive/project/aks/viya4-iac-azure/gel-vars.tfvars
  ```

* It defines the node pools and what kind of Azure Instance sizes will be used for the Kubernetes Workers

### Generate a Terraform plan

* Let's generate the TF plan corresponding to the AKS cluster with multile node pools :

  ```bash
  # generate the TF plan corresponding to the AKS cluster with multiple node pools
  cd ~/clouddrive/project/aks/viya4-iac-azure
  #terraform plan -input=false \
  $HOME/bin/terraform plan -input=false \
      -var-file=./gel-vars.tfvars \
      -out ./my-aks.plan
  ```

* Now you should see something like that :

  ![tf plan](img/2020-11-19-10-22-20.png)

* Review the generated plan

  ```sh
  #TFPLAN=my-aks-minimal.plan
  TFPLAN=my-aks.plan
  cd ~/clouddrive/project/aks/viya4-iac-azure
  #terraform show ${TFPLAN}
  $HOME/bin/terraform show ${TFPLAN}
  ```

* The plan shows all the infrastructure components that will be created in Azure to support the Viya deployment : Virtual network, Subnets, Public IPs, Network Security Groups, an Azure Postgres DB, an Azure Kubernetes Cluster with several node pools and associated instance types.
* A first resource group named per your SAS ID or NAME (```YOUR SAS ID```viya4aks-rg) is created to hold all the provisioned components.
* Then when the AKS cluster is created in the first Resource group, it will trigger the creation of the second resource group (the name starts with "MC" and also contains your SAS ID) that will contains all the Kubernetes cluster infrastructure (VM scalesets, Disks, Storage accounts, Public IP, etc...)

## Stand up AKS cluster and Validate

### Apply the Terraform plan

* Now let's deploy the AKS cluster with the TF plan
* In our testing it takes between 7 to 8 minutes.

    ```bash
    # Deploy the AKS cluster with the TF plan
    TFPLAN=my-aks.plan
    # by default, we go with the multi node pools AKS cluster but you can choose the minimal one to test
    cd ~/clouddrive/project/aks/viya4-iac-azure
    # apply the TS plan and generate a log in case of disconnection.
    time $HOME/bin/terraform apply "./${TFPLAN}" 2>&1 \
    | tee -a /tmp/terraform-apply.log
    ```

* Now, you can see the progress of the Terraform plan being executed to provision the AKS infrastructure.

    ![TF run](img/2020-07-22-10-38-56.png)

* You can also notice in the Azure Portal that the new resource groups (the primary and the one generated to hold the AKS cluster infrastructure, named "MCxxx") start to get populated.

  ![aks deployment](img/2020-09-10-10-29-41.png)

  ![aks deployment MC](img/2020-09-10-10-30-26.png)

* Finally, in the Cloud Shell window, you should see some green text reporting that the TF plan run is now complete.

  ![TF completed](img/2020-11-19-10-39-40.png)

* Browse in the portal to see all resources created in the resource group.
* An additional Resource group has been created with the infrastrcuture components of the AKS cluster (disks, load-balancer, etc...).

  _Note: If you don't see your resource group, you might not be looking in the correct subscription. If you don't see the sas-gelsandbox subscription you might have to change the global subscription filter in the "Subscription" menu_

  ![MC resource group](img/2020-07-24-17-11-06.png)

### In case it did not work for you

* It is possible that the AKS cluster creation failed and that you see some errors, for example :

    ```log
    "Error: Error running command 'kubectl --kubeconfig "xxxxxxviya4aks-aks-kubeconfig.conf" apply -f ./sas_iac_buildinfo.yaml
    ': exit status 1. Output: Unable to connect to the server: dial tcp 20.72.152.72:443: i/o timeout"
    ```

* This issue is likely caused by my cloud shell timing out.
* When you log back in to the cloud shell you now have a different external IP.
* You need to reflect this in the gel-vars.tfvars TF variables file.
* If Terraform created a terraform.tfstate file from a previous run then delete this file
* Then restart from this [step](#customize-the-terraform-variables) to reset the CLOUDSHELLIP env and edit the gel-vars.tfvars to include the new ip. Then repeat the TF build and apply steps.
* Throughout these labs be cognizant of when your cloud shell times out. In some cases depending upon where you are in the labs you will need to factor in the new ip address for CLOUDSHELLIP.

### Getting the kubectl config file for your newly built AKS cluster

* First, let's generate the config file with a recognizable name:

  ```bash
  # generate the config file with a recognizable name
  cd ~/clouddrive/project/aks/viya4-iac-azure
  mkdir -p ~/.kube
  #terraform output kube_config > ~/.kube/${STUDENT}-aks-kubeconfig.conf
  $HOME/bin/terraform output kube_config > ~/.kube/${STUDENT}-aks-kubeconfig.conf
  ```

* Then, let's make a symlink to it for easier access:

  ```bash
  SOURCEFOLDER=~/.kube/${STUDENT}-aks-kubeconfig.conf
  ansible localhost -m file \
    -a "src=$SOURCEFOLDER \
        dest=~/.kube/config state=link" \
    --diff

  ```

* If you have the application "Lens" on your desktop, and if you are VPN'ed directly to Cary, you can use that config file to configure Lens.

* Now run the kubectl command to check if your cluster is there !

  ```bash
  kubectl get nodes

  ```

* You should see something like that :

    ```log
    NAME                              STATUS   ROLES   AGE     VERSION
    aks-cas-22223626-vmss000000       Ready    agent   79s     v1.18.10
    aks-compute-22223626-vmss000000   Ready    agent   112s    v1.18.10
    aks-system-22223626-vmss000000    Ready    agent   5m52s   v1.18.10
    ```

_Note : Don't worry if you don't see more nodes (for example stateful and stateless), they will be dynamically provisionned by the cluster auto-scaler during the deployment._

### Disable the authorized IP range for the Kubernetes API

We have run into issues where timeouts in the Cloud Shell effectively locked us out of the Kubernetes cluster.
Therefore, instead of using the --api-server-authorized-ip-ranges in Kubernetes, it's a Security Group (firewall) that will secure our kubectl access.

reference : <https://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges#update-or-disable-authorized-ip-ranges>

* Here is the command you should run:

  ```bash
  az aks update -n ${STUDENT}viya4aks-aks \
      -g ${STUDENT}viya4aks-rg --api-server-authorized-ip-ranges ""

  ```

### Configure kubectl auto-completion

The Kubernetes command-line tool, `kubectl`, allows you to run commands against Kubernetes clusters. You can use `kubectl` to deploy applications, inspect and manage cluster resources, and view logs.

`kubectl` provides autocompletion support for Bash and Zsh, which can save you a lot of typing.

* Run the command below to enable it in your cloud shell environment:

  ```bash
  source <(kubectl completion bash)
  ansible localhost \
      -m lineinfile \
      -a "dest=~/.bashrc \
          line='source <(kubectl completion bash)' \
          state=present" \
      --diff
  ```

## IMPORTANT FOR THE NEXT TIME : DO NOT REDO EVERYTHING

**Most of the steps that you have performed until now will not be required the next time that you will perform this hands-on to deploy Viya in AKS.**

If you delete your AKS cluster and close your Cloud Shell session now, you will be able to use the cheatcodes [there](11_099_Fast_track_with_cheatcodes.md) to rebuild automatically your AKS cluster as defined in the Hands-on.

It is because most of the changes and configuration will be persisted by Azure (the "clouddrive" folder and the "home folder" are kept), and the next time you will start a new Cloud Shell you won't have to redo them. The only real change will be the Cloud Shell IP address.

<!-- ## FAST TRACK (REBUILD THE AKS CLUSTER FROM A NEW CLOUD SHELL SESSION)

If you deleted your first AKS cluster and wants to re-create without having to redo everything from scratch, just follow the steps below.
We assume that the Terraform plan is already in place and the only change is the client IP address.

```sh
# Configure az CLI to use our subscription and eastus region by default
az account set -s "sas-gelsandbox"
az configure --defaults location=eastus
# Reset the IDs (Service Principals) we need for Terraform
cd ~/clouddrive
. ./TF_CLIENT_CREDS
# Get the CloudShell container public IP
myPubIP=$(curl https://ifconfig.me)
# Update the TF build plan access whitelist with our public IP
ansible localhost -m lineinfile -a "dest=~/clouddrive/project/aks/viya4-iac-azure/gel-vars.tfvars backup='yes' regexp='^cluster_endpoint_public_access_cidrs' line='cluster_endpoint_public_access_cidrs = [\"109.232.56.224/27\", \"149.173.0.0/16\", \"194.206.69.176/28\", \"$myPubIP/32\"]'" --diff
# Rebuild the TF plan with the new IP address so we can get kubectl access
cd ~/clouddrive/project/aks/viya4-iac-azure
$HOME/bin/terraform plan -input=false \
     -var-file=./gel-vars.tfvars \
     -out ./my-aks.plan
# Apply the TF plan (deploy the AKS cluster...)
time $HOME/bin/terraform apply "./my-aks.plan"
# Get the Kubectl config file and set KUBECONFIG
$HOME/bin/terraform output kube_config > ~/.kube/$(cat ~/student.txt)-aks-kubeconfig.conf
SOURCEFOLDER=~/.kube/${STUDENT}-aks-kubeconfig.conf
ansible localhost -m file -a "src=$SOURCEFOLDER dest=~/.kube/config state=link" --diff
# Check if we can see the K8s nodes
kubectl get nodes
## Disable K8s authorized API Server range (in case our pulic IP changes again !)
az aks update -n ${STUDENT}viya4aks-aks -g ${STUDENT}viya4aks-rg --api-server-authorized-ip-ranges ""
``` -->

## FAST TRACK (RECONNECT TO THE AKS CLUSTER WITH A NEW IP)

If the AKS cluster remains there but you just get a new Cloud Shell session, you should be able to connect to it even though your local IP has changed.

```sh
kubectl get nodes

```

## References

* TF and Cloud-Shell : <https://docs.microsoft.com/en-us/azure/developer/terraform/get-started-cloud-shell>
* AKS cluster auto-scaling: <https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler>

## Navigation

<!-- startnav -->
* [01 Introduction / 01 031 Booking a Lab Environment for the Workshop](/01_Introduction/01_031_Booking_a_Lab_Environment_for_the_Workshop.md)
* [01 Introduction / 01 032 Assess Readiness of Lab Environment](/01_Introduction/01_032_Assess_Readiness_of_Lab_Environment.md)
* [01 Introduction / 01 033 CheatCodes](/01_Introduction/01_033_CheatCodes.md)
* [02 Kubernetes and Containers Fundamentals / 02 131 Learning about Namespaces](/02_Kubernetes_and_Containers_Fundamentals/02_131_Learning_about_Namespaces.md)
* [03 Viya 4 Software Specifics / 03 011 Looking at a Viya 4 environment with Visual Tools DEMO](/03_Viya_4_Software_Specifics/03_011_Looking_at_a_Viya_4_environment_with_Visual_Tools_DEMO.md)
* [03 Viya 4 Software Specifics / 03 051 Create your own Viya order](/03_Viya_4_Software_Specifics/03_051_Create_your_own_Viya_order.md)
* [03 Viya 4 Software Specifics / 03 056 Getting the order with the CLI](/03_Viya_4_Software_Specifics/03_056_Getting_the_order_with_the_CLI.md)
* [04 Pre Requisites / 04 081 Pre Requisites automation with Viya4-ARK](/04_Pre-Requisites/04_081_Pre-Requisites_automation_with_Viya4-ARK.md)
* [05 Deployment tools / 05 121 Setup a Windows Client Machine](/05_Deployment_tools/05_121_Setup_a_Windows_Client_Machine.md)
* [06 Deployment Steps / 06 031 Deploying a simple environment](/06_Deployment_Steps/06_031_Deploying_a_simple_environment.md)
* [06 Deployment Steps / 06 051 Deploying Viya with Authentication](/06_Deployment_Steps/06_051_Deploying_Viya_with_Authentication.md)
* [06 Deployment Steps / 06 061 Deploying in a second namespace](/06_Deployment_Steps/06_061_Deploying_in_a_second_namespace.md)
* [06 Deployment Steps / 06 071 Removing Viya deployments](/06_Deployment_Steps/06_071_Removing_Viya_deployments.md)
* [06 Deployment Steps / 06 081 Deploying a programing only environment](/06_Deployment_Steps/06_081_Deploying_a_programing-only_environment.md)
* [07 Deployment Customizations / 07 021 Configuring SASWORK](/07_Deployment_Customizations/07_021_Configuring_SASWORK.md)
* [07 Deployment Customizations / 07 051 Adding a local registry to k8s](/07_Deployment_Customizations/07_051_Adding_a_local_registry_to_k8s.md)
* [07 Deployment Customizations / 07 052 Using mirror manager to populate the local registry](/07_Deployment_Customizations/07_052_Using_mirror_manager_to_populate_the_local_registry.md)
* [07 Deployment Customizations / 07 053 Deploy from local registry](/07_Deployment_Customizations/07_053_Deploy_from_local_registry.md)
* [07 Deployment Customizations / 07 091 Configure SAS ACCESS Engine](/07_Deployment_Customizations/07_091_Configure_SAS_ACCESS_Engine.md)
* [07 Deployment Customizations / 07 101 Configure SAS ACCESS TO HADOOP](/07_Deployment_Customizations/07_101_Configure_SAS_ACCESS_TO_HADOOP.md)
* [07 Deployment Customizations / 07 102 Parralel loading with EP for Hadoop](/07_Deployment_Customizations/07_102_Parralel_loading_with_EP_for_Hadoop.md)
* [11 Azure AKS Deployment / 11 011 Creating an AKS Cluster](/11_Azure_AKS_Deployment/11_011_Creating_an_AKS_Cluster.md)**<-- you are here**
* [11 Azure AKS Deployment / 11 041 Performing Prereqs in AKS](/11_Azure_AKS_Deployment/11_041_Performing_Prereqs_in_AKS.md)
* [11 Azure AKS Deployment / 11 042 Deploying Viya 4 on AKS](/11_Azure_AKS_Deployment/11_042_Deploying_Viya_4_on_AKS.md)
* [11 Azure AKS Deployment / 11 043 Deploy a second namespace in AKS](/11_Azure_AKS_Deployment/11_043_Deploy_a_second_namespace_in_AKS.md)
* [11 Azure AKS Deployment / 11 051 CAS Customizations](/11_Azure_AKS_Deployment/11_051_CAS_Customizations.md)
* [11 Azure AKS Deployment / 11 061 Install monitoring and logging](/11_Azure_AKS_Deployment/11_061_Install_monitoring_and_logging.md)
* [11 Azure AKS Deployment / 11 091 Deleting the AKS Cluster](/11_Azure_AKS_Deployment/11_091_Deleting_the_AKS_Cluster.md)
* [11 Azure AKS Deployment / 11 099 Fast track with cheatcodes](/11_Azure_AKS_Deployment/11_099_Fast_track_with_cheatcodes.md)
<!-- endnav -->
