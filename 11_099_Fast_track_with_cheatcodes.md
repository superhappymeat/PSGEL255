![Global Enablement & Learning](https://gelgitlab.race.sas.com/GEL/utilities/writing-content-in-markdown/-/raw/master/img/gel_banner_logo_tech-partners.jpg)
# Fast track with the cheatcodes (WORK IN PROGRESS)

* [WARNING](#warning)
* [Extract the latest payload archive and generate the cheatcodes](#extract-the-latest-payload-archive-and-generate-the-cheatcodes)
* [Run the cheatcodes](#run-the-cheatcodes)
* [Navigation](#navigation)

## WARNING

You can use the cheatcodes for everything EXCEPT for the very first manual steps which are :
    -  open and configure the Azure Cloud Shell
    -  upload the "payload" archive (from there: <https://gelweb.race.sas.com/scripts/PSGEL255/payload/payload.tgz> ) in the Azure Shell.

See instructions in this [page](11_011_Creating_an_AKS_Cluster.md) if needed.

Once you are logged into Cloud Shell and have uploaded the payload archive, you can start to use the cheatcodes to perform the provisionning and deployment.

The cheatcodes are built from md files that are provided as part of the payload:

## Extract the latest payload archive and generate the cheatcodes

```sh
rm -Rf ~/clouddrive/payload
cp ~/payload*.tgz ~/clouddrive/payload.tgz
cd ~/clouddrive
tar -xvf payload.tgz && rm payload.tgz
# if you need to checkout from a specific branch/Viya release:
# cd ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1-on-kubernetes
# BRANCH=stable-2020.1.1;
# git checkout --force ${BRANCH}
bash ~/clouddrive/payload/cheatcodes/create.cheatcodes.sh ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1-on-kubernetes/11_Azure_AKS_Deployment/
```

Now you can directly call the cheatcodes for each step.

## Run the cheatcodes

* To build the AKS cluster, perform the pre-requisites and deploy Viya 4

    ```sh
    bash -x ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1*/11_Azure_AKS_*/11_011_Creating_an_AKS_Cluster.sh 2>&1 \
    | tee -a ~/clouddrive/11_011_Creating_an_AKS_Cluster.log
    bash -x ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1*/11_Azure_AKS_*/11_041_Performing_Prereqs_in_AKS.sh 2>&1 \
    | tee -a ~/clouddrive/11_041_Performing_Prereqs_in_AKS.log
    bash -x ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1*/11_Azure_AKS_*/11_042_Deploying_Viya_4_on_AKS.sh 2>&1 \
    | tee -a ~/clouddrive/11_042_Deploying_Viya_4_on_AKS.log
    ```

    Important: it is strongly recommended to send the output into a log file, in case of a disconnection from the Cloud Shell environment (even if you get disconnected, when you reconnect, you will likely recover the Cloud Shell container and see that the cheat codes continue to run).

* Optionaly you can run this command to install the

    ```sh
    bash -x /usr/csuser/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1*/11_Azure_AKS_*/11_061_Install_monitoring_and_logging.sh 2>&1 \
    | tee -a ~/clouddrive/11_061_Install_monitoring_and_logging.log
    ```


* Or if you want to run all the hands-on of the section automatically, simply run :

    ```sh
    time bash -x ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1*/11_Azure_AKS_*/_all.sh 2>&1 \
    | tee -a ~/clouddrive/_all.log
    ```

    Note that this one will also run the "Install monitoring and logging" hands-on instructions.

* Finally you can also use tmux, if you prefer

    ```sh
    cd ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1-on-kubernetes/
    tmux new-session ' time bash -x ./11_Azure_AKS_Deployment//_all.sh | tee ./11_Azure_AKS_Deployment//_all.log ; bash '
    ```

    In case you get disconnected in the middle of the tmux session, you might be able to reconnect to it after reconnecting to your Azure Shell session.

    ```sh
    tmux ls
    tmux a -t <tmux session id>
    ```

    Otherwise just tail the log.

    ```sh
    cd ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1-on-kubernetes/
    tail -f ./11_Azure_AKS_Deployment//_all.log
    ```

<!-- DON'T DO THE STEPS BELOW

Process to test cheatcode :

* Commit and push to branch
* Rebuild the payload (from jenkins)
http://gelkins.race.sas.com:8080/job/Viya%204%20Deployment%20Workshop/job/115%20-%20Create%20Deployment%20Payload/
click on "build with parameter".
then "build"

* Reupload the payload in Cloud Shell : https://gelweb.race.sas.com/scripts/PSGEL255/payload/payload.tgz
* Replace the old payload:

```sh
rm -Rf ~/clouddrive/payload
mv ~/payload*.tgz ~/clouddrive/payload.tgz
cd ~/clouddrive
tar -xvf payload.tgz && rm payload.tgz
```

* Build cheatcodes

```sh
cd ~/clouddrive/payload/workshop/PSGEL255-deploying-viya-4.0.1-on-kubernetes/
# Temporary : until it works well we need to checkout from the working branch
# BRANCH=master
BRANCH=aks-tf-part2
git checkout --force ${BRANCH}
bash ~/clouddrive/payload/cheatcodes/create.cheatcodes.sh ./11_Azure_AKS_Deployment/
```

* Test them ONE BY ONE -->

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
* [11 Azure AKS Deployment / 11 011 Creating an AKS Cluster](/11_Azure_AKS_Deployment/11_011_Creating_an_AKS_Cluster.md)
* [11 Azure AKS Deployment / 11 041 Performing Prereqs in AKS](/11_Azure_AKS_Deployment/11_041_Performing_Prereqs_in_AKS.md)
* [11 Azure AKS Deployment / 11 042 Deploying Viya 4 on AKS](/11_Azure_AKS_Deployment/11_042_Deploying_Viya_4_on_AKS.md)
* [11 Azure AKS Deployment / 11 043 Deploy a second namespace in AKS](/11_Azure_AKS_Deployment/11_043_Deploy_a_second_namespace_in_AKS.md)
* [11 Azure AKS Deployment / 11 051 CAS Customizations](/11_Azure_AKS_Deployment/11_051_CAS_Customizations.md)
* [11 Azure AKS Deployment / 11 061 Install monitoring and logging](/11_Azure_AKS_Deployment/11_061_Install_monitoring_and_logging.md)
* [11 Azure AKS Deployment / 11 091 Deleting the AKS Cluster](/11_Azure_AKS_Deployment/11_091_Deleting_the_AKS_Cluster.md)
* [11 Azure AKS Deployment / 11 099 Fast track with cheatcodes](/11_Azure_AKS_Deployment/11_099_Fast_track_with_cheatcodes.md)**<-- you are here**
<!-- endnav -->
