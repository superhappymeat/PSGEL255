![Global Enablement & Learning](https://gelgitlab.race.sas.com/GEL/utilities/writing-content-in-markdown/-/raw/master/img/gel_banner_logo_tech-partners.jpg)

* [Configure Azure autoscaling to 0 nodes](#configure-azure-autoscaling-to-0-nodes)
  * [Stop and then restart the environment](#stop-and-then-restart-the-environment)
    * [Commands to scale down to 0 the Viya environment (STOP)](#commands-to-scale-down-to-0-the-viya-environment-stop)
    * [Commands to scale up the Viya environment (RESTART)](#commands-to-scale-up-the-viya-environment-restart)

# Stop-shrink and Start-scale in AKS

## Configure Azure autoscaling to 0 nodes

* To reduce the costs we would like to set the min_count for our SAS Viya Nodepools to 0. So when you stop the Viya environment, the nodes should disappear after a a little while, saving significant costs.

* The easiest way would have been to set it in the Terraform variables. However a bug is preventing the min_count/node_count to be set to zero in TF: <https://github.com/terraform-providers/terraform-provider-azurerm/pull/8300>.

* In addition portal does not allow you to it. So we use the az CLI to do it.

    ```bash
    az aks nodepool update --update-cluster-autoscaler \
    --min-count 0 --max-count 3 -g ${STUDENT}viya4aks-rg \
    -n stateless --cluster-name ${STUDENT}viya4aks-aks

    az aks nodepool update --update-cluster-autoscaler \
    --min-count 0 --max-count 3 -g ${STUDENT}viya4aks-rg \
    -n stateful --cluster-name ${STUDENT}viya4aks-aks

    az aks nodepool update --update-cluster-autoscaler \
    --min-count 0 --max-count 1 -g ${STUDENT}viya4aks-rg \
    -n compute --cluster-name ${STUDENT}viya4aks-aks

    az aks nodepool update --update-cluster-autoscaler \
    --min-count 0 --max-count 8 -g ${STUDENT}viya4aks-rg \
    -n cas --cluster-name ${STUDENT}viya4aks-aks

    az aks nodepool update --update-cluster-autoscaler \
    --min-count 0 --max-count 1 -g ${STUDENT}viya4aks-rg \
    -n connect --cluster-name ${STUDENT}viya4aks-aks
    ```

_Note: depending on the initial node pool configuration, some of the command above might fail and show an error when the autoscaler is not enabled for the node pool (when min-count=max-count). You can safely ignore the errors, it just means that these node pools will not scale down to 0 when we stop the Viya services._

### Stop and then restart the environment

Follow the documented instructions to [stop your environment](https://go.documentation.sas.com/?cdcId=itopscdc&cdcVersion=v_001LTS&docsetId=itopssrv&docsetTarget=n0pwhguy22yhe0n1d7pgi63mf6pb.htm&locale=en#p1nz12w805sz14n1rd6m593qnefy).

Once it's fully stopped, use the other instructions to [restart it](https://go.documentation.sas.com/?cdcId=itopscdc&cdcVersion=v_001LTS&docsetId=itopssrv&docsetTarget=n0pwhguy22yhe0n1d7pgi63mf6pb.htm&locale=en#p0szdr59qn1uwkn13ktqmzqjwpyh).

Because we are nice, instead of letting you struggle with the documentation, we prepared the commands for you below in our Lab environment. However be aware that the currently documentated process to do it is manual,cumbersome and error prone.

#### Commands to scale down to 0 the Viya environment (STOP)

* Add a new transformer for the scale down to 0 of some of the resources (phase 0)

    ```bash
    # Add a new transformer
    printf "
    - command: update
      path: transformers[+]
      value:
        sas-bases/overlays/scaling/zero-scale/phase-0-transformer.yaml
    " | $HOME/bin/yq -I 4 w -i -s - ~/clouddrive/project/deploy/test/kustomization.yaml
    ```

* Rebuild the manifest

    ```bash
    cd ~/clouddrive/project/deploy/test
    kustomize build -o site.yaml
    ```

* Apply the updated site.yaml file to your deployment:

    ```bash
    kubectl apply -f site.yaml
    ```

* Wait for CAS operator-managed pods to get deleted:

    ```sh
    kubectl -n test wait --for=delete -l casoperator.sas.com/server=default pods
    ```

* Add an other new transformer for the scale down to 0 of the rest of the resources (phase 1)

    ```bash
    # Add a new transformer
    printf "
    - command: update
      path: transformers[+]
      value:
        sas-bases/overlays/scaling/zero-scale/phase-1-transformer.yaml
    " | $HOME/bin/yq -I 4 w -i -s - ~/clouddrive/project/deploy/test/kustomization.yaml
    ```

* Rebuild the manifest

    ```bash
    cd ~/clouddrive/project/deploy/test
    kustomize build -o site.yaml
    ```

* Apply the updated site.yaml file to your deployment:

    ```bash
    kubectl apply -f site.yaml
    ```

#### Commands to scale up the Viya environment (RESTART)

* Remove the 2 lines, rebuild and reapply.

    ```bash
    $HOME/bin/yq d -i kustomization.yaml 'transformers(.==sas-bases/overlays/scaling/zero-scale/phase-0-transformer.yaml)'
    $HOME/bin/yq d -i kustomization.yaml 'transformers(.==sas-bases/overlays/scaling/zero-scale/phase-1-transformer.yaml)'
    ```

* Rebuild the manifest

    ```bash
    cd ~/clouddrive/project/deploy/test
    kustomize build -o site.yaml
    ```

* Apply the updated site.yaml file to your deployment:

    ```sh
    kubectl apply -f site.yaml
    ```

    _Note: if you run this hands-on exercise with the cheatcodes, it will just stop (scale down to 0) the Viya environment but NOT restart it, so you can also witness the AKS autoscaler behavior. To restart it, just reapply the site.yaml manifest_

<!-- OLD WAY
* Stop all the "running" pods for good, you would have to run:

    ```sh
    kubectl -n test scale deployments --all --replicas=0
    kubectl -n test scale statefulsets --all --replicas=0
    kubectl -n test delete casdeployment --all
    kubectl -n test delete jobs --all
    ```

* Note that if we want to restart at this point you can run :

    ```sh
    cd ~/clouddrive/project/deploy/test
    kubectl -n test apply -f site.yaml
    ```

  But because we've done a scale to zero, and because the number of replicas is not part of the manifest, we also have to do a scale to 1 !

    ```sh
    kubectl -n test scale deployments --all --replicas=1
    kubectl -n test scale statefulsets --all --replicas=1
    ```
-->