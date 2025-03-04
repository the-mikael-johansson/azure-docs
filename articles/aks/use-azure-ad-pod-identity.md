---
title: Use Azure Active Directory pod-managed identities in Azure Kubernetes Service (Preview)
description: Learn how to use AAD pod-managed managed identities in Azure Kubernetes Service (AKS)
services: container-service
ms.topic: article
ms.date: 3/12/2021

---

# Use Azure Active Directory pod-managed identities in Azure Kubernetes Service (Preview)

Azure Active Directory pod-managed identities uses Kubernetes primitives to associate [managed identities for Azure resources][az-managed-identities] and identities in Azure Active Directory (AAD) with pods. Administrators create identities and bindings as Kubernetes primitives that allow pods to access Azure resources that rely on AAD as an identity provider.

> [!NOTE]
>The feature described in this document, pod-managed identities (preview), will be replaced with pod-managed identities V2 (preview).
> If you have an existing installation of AADPODIDENTITY, there will be a migration option to V2. More details on the migration will follow as we get closer to Public Preview slated for Q2 2022. Enabling this feature means that the MIC component isn't needed.

[!INCLUDE [preview features callout](./includes/preview/preview-callout.md)]

## Before you begin

You must have the following resource installed:

* The Azure CLI, version 2.20.0 or later
* The `aks-preview` extension version 0.5.5 or later

### Limitations

* A maximum of 200 pod identities are allowed for a cluster.
* A maximum of 200 pod identity exceptions are allowed for a cluster.
* Pod-managed identities are available on Linux node pools only.

### Register the `EnablePodIdentityPreview`

Register the `EnablePodIdentityPreview` feature:

```azurecli
az feature register --name EnablePodIdentityPreview --namespace Microsoft.ContainerService
```

### Install the `aks-preview` Azure CLI

You also need the *aks-preview* Azure CLI extension version 0.5.5 or later. Install the *aks-preview* Azure CLI extension by using the [az extension add][az-extension-add] command. Or install any available updates by using the [az extension update][az-extension-update] command.

```azurecli-interactive
# Install the aks-preview extension
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview
```

## Create an AKS cluster with Azure Container Networking Interface (CNI)

> [!NOTE]
> This is the default recommended configuration

Create an AKS cluster with Azure CNI and pod-managed identity enabled. The following commands use [az group create][az-group-create] to create a resource group named *myResourceGroup* and the [az aks create][az-aks-create] command to create an AKS cluster named *myAKSCluster* in the *myResourceGroup* resource group.

```azurecli-interactive
az group create --name myResourceGroup --location eastus
az aks create -g myResourceGroup -n myAKSCluster --enable-pod-identity --network-plugin azure
```
> [!NOTE]
> Azure Active Directory Pod Identity supports 2 modes of operation:
> 
> 1. Standard Mode: In this mode, the following 2 components are deployed to the AKS cluster: 
>     * [Managed Identity Controller(MIC)](https://azure.github.io/aad-pod-identity/docs/concepts/mic/): A Kubernetes controller that watches for changes to pods, [AzureIdentity](https://azure.github.io/aad-pod-identity/docs/concepts/azureidentity/) and [AzureIdentityBinding](https://azure.github.io/aad-pod-identity/docs/concepts/azureidentitybinding/) through the Kubernetes API Server. When it detects a relevant change, the MIC adds or deletes [AzureAssignedIdentity](https://azure.github.io/aad-pod-identity/docs/concepts/azureassignedidentity/) as needed. Specifically, when a pod is scheduled, the MIC assigns the managed identity on Azure to the underlying VMSS used by the node pool during the creation phase. When all pods using the identity are deleted, it removes the identity from the VMSS of the node pool, unless the same managed identity is used by other pods. The MIC takes similar actions when AzureIdentity or AzureIdentityBinding are created or deleted.
>     * [Node Managed Identity (NMI)](https://azure.github.io/aad-pod-identity/docs/concepts/nmi/): is a pod that runs as a DaemonSet on each node in the AKS cluster. NMI intercepts security token requests to the [Azure Instance Metadata Service](../virtual-machines/linux/instance-metadata-service.md?tabs=linux) on each node, redirect them to itself and validates if the pod has access to the identity it's requesting a token for and fetch the token from the Azure Active Directory tenant on behalf of the application.
> 2. Managed Mode: In this mode, there is only NMI. The identity needs to be manually assigned and managed by the user. For more information, see [Pod Identity in Managed Mode](https://azure.github.io/aad-pod-identity/docs/configure/pod_identity_in_managed_mode/).
>
>When you install the Azure Active Directory Pod Identity via Helm chart or YAML manifest as shown in the [Installation Guide](https://azure.github.io/aad-podidentity/docs/getting-started/installation/), you can choose between the `standard` and `managed` mode. If you instead decide to install the Azure Active Directory Pod Identity using the [AKS cluster add-on]() as shown in this article, the setup will use the `managed` mode.

Use [az aks get-credentials][az-aks-get-credentials] to sign in to your AKS cluster. This command also downloads and configures the `kubectl` client certificate on your development computer.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## Update an existing AKS cluster with Azure CNI

Update an existing AKS cluster with Azure CNI to include pod-managed identity.

```azurecli-interactive
az aks update -g $MY_RESOURCE_GROUP -n $MY_CLUSTER --enable-pod-identity
```
## Using Kubenet network plugin with Azure Active Directory pod-managed identities 

> [!IMPORTANT]
> Running aad-pod-identity in a cluster with Kubenet is not a recommended configuration because of the security implication. Please follow the mitigation steps and configure policies before enabling aad-pod-identity in a cluster with Kubenet.

## Mitigation

To mitigate the vulnerability at the cluster level, you can use the Azure built-in policy "Kubernetes cluster containers should only use allowed capabilities" to limit the CAP_NET_RAW attack.  

Add NET_RAW to "Required drop capabilities"

![image](https://user-images.githubusercontent.com/50749048/118558790-206b8880-b735-11eb-9e48-236b81116812.png)

If you are not using Azure Policy, you can use OpenPolicyAgent admission controller together with Gatekeeper validating webhook. Provided you have Gatekeeper already installed in your cluster, add the ConstraintTemplate of type K8sPSPCapabilities:

```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/capabilities/template.yaml
```
Add a template to limit the spawning of Pods with the NET_RAW capability:

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPCapabilities
metadata:
  name: prevent-net-raw
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - "kube-system"
  parameters:
    requiredDropCapabilities: ["NET_RAW"]
```

## Create an AKS cluster with Kubenet network plugin

Create an AKS cluster with Kubenet network plugin and pod-managed identity enabled.

```azurecli-interactive
az aks create -g $MY_RESOURCE_GROUP -n $MY_CLUSTER --enable-pod-identity --enable-pod-identity-with-kubenet
```

## Update an existing AKS cluster with Kubenet network plugin

Update an existing AKS cluster with Kubenet network plugin to include pod-managed identity.

```azurecli-interactive
az aks update -g $MY_RESOURCE_GROUP -n $MY_CLUSTER --enable-pod-identity --enable-pod-identity-with-kubenet
```

## Create an identity

Create an identity using [az identity create][az-identity-create] and set the *IDENTITY_CLIENT_ID* and *IDENTITY_RESOURCE_ID* variables.

```azurecli-interactive
az group create --name myIdentityResourceGroup --location eastus
export IDENTITY_RESOURCE_GROUP="myIdentityResourceGroup"
export IDENTITY_NAME="application-identity"
az identity create --resource-group ${IDENTITY_RESOURCE_GROUP} --name ${IDENTITY_NAME}
export IDENTITY_CLIENT_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME} --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME} --query id -otsv)"
```

## Assign permissions for the managed identity

To run the demo, the *IDENTITY_CLIENT_ID* managed identity must have Virtual Machine Contributor permissions in the resource group that contains the virtual machine scale set of your AKS cluster.

```azurecli-interactive
NODE_GROUP=$(az aks show -g myResourceGroup -n myAKSCluster --query nodeResourceGroup -o tsv)
NODES_RESOURCE_ID=$(az group show -n $NODE_GROUP -o tsv --query "id")
az role assignment create --role "Virtual Machine Contributor" --assignee "$IDENTITY_CLIENT_ID" --scope $NODES_RESOURCE_ID
```

## Create a pod identity

Create a pod identity for the cluster using `az aks pod-identity add`.

> [!IMPORTANT]
> You must have the relevant permissions (for example, Owner) on your subscription to create the identity and assign role binding to the cluster identity.
> 
> The cluster identity must have Managed Identity Operator permissions for the identity to be assigned.

```azurecli-interactive
export POD_IDENTITY_NAME="my-pod-identity"
export POD_IDENTITY_NAMESPACE="my-app"
az aks pod-identity add --resource-group myResourceGroup --cluster-name myAKSCluster --namespace ${POD_IDENTITY_NAMESPACE}  --name ${POD_IDENTITY_NAME} --identity-resource-id ${IDENTITY_RESOURCE_ID}
```

> [!NOTE]
> When you enable pod-managed identity on your AKS cluster, an AzurePodIdentityException named *aks-addon-exception* is added to the *kube-system* namespace. An AzurePodIdentityException allows pods with certain labels to access the Azure Instance Metadata Service (IMDS) endpoint without being intercepted by the node-managed identity (NMI) server. The *aks-addon-exception* allows AKS first-party addons, such as AAD pod-managed identity, to operate without having to manually configure an AzurePodIdentityException. Optionally, you can add, remove, and update an AzurePodIdentityException using `az aks pod-identity exception add`, `az aks pod-identity exception delete`, `az aks pod-identity exception update`, or `kubectl`.

> [!NOTE]
> When you assign the pod identity by using `pod-identity add`, the Azure CLI attempts to grant the Managed Identity Operator role over the pod identity (*IDENTITY_RESOURCE_ID*) to the cluster identity.

## Run a sample application

For a pod to use AAD pod-managed identity, the pod needs an *aadpodidbinding* label with a value that matches a selector from a *AzureIdentityBinding*. To run a sample application using AAD pod-managed identity, create a `demo.yaml` file with the following contents. Replace *POD_IDENTITY_NAME*, *IDENTITY_CLIENT_ID*, and *IDENTITY_RESOURCE_GROUP* with the values from the previous steps. Replace *SUBSCRIPTION_ID* with your subscription ID.

> [!NOTE]
> In the previous steps, you created the *POD_IDENTITY_NAME*, *IDENTITY_CLIENT_ID*, and *IDENTITY_RESOURCE_GROUP* variables. You can use a command such as `echo` to display the value you set for variables, for example `echo $IDENTITY_NAME`.

```yml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    aadpodidbinding: $POD_IDENTITY_NAME
spec:
  containers:
  - name: demo
    image: mcr.microsoft.com/oss/azure/aad-pod-identity/demo:v1.6.3
    args:
      - --subscriptionid=$SUBSCRIPTION_ID
      - --clientid=$IDENTITY_CLIENT_ID
      - --resourcegroup=$IDENTITY_RESOURCE_GROUP
    env:
      - name: MY_POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: MY_POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: MY_POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
  nodeSelector:
    kubernetes.io/os: linux
```

Notice the pod definition has an *aadpodidbinding* label with a value that matches the name of the pod identity you ran `az aks pod-identity add` in the previous step.

Deploy `demo.yaml` to the same namespace as your pod identity using `kubectl apply`:

```azurecli-interactive
kubectl apply -f demo.yaml --namespace $POD_IDENTITY_NAMESPACE
```

Verify the sample application successfully runs using `kubectl logs`.

```azurecli-interactive
kubectl logs demo --follow --namespace $POD_IDENTITY_NAMESPACE
```

Verify the logs show the a token is successfully acquired and the *GET* operation is successful.
 
```output
...
successfully doARMOperations vm count 0
successfully acquired a token using the MSI, msiEndpoint(http://169.254.169.254/metadata/identity/oauth2/token)
successfully acquired a token, userAssignedID MSI, msiEndpoint(http://169.254.169.254/metadata/identity/oauth2/token) clientID(xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
successfully made GET on instance metadata
...
```
## Run an application with multiple identities

## Create multiple identities

Create identities using [az identity create][az-identity-create] and set the *IDENTITY_CLIENT_ID* and *IDENTITY_RESOURCE_ID* variables.

```azurecli-interactive
az group create --name myIdentityResourceGroup --location eastus
export IDENTITY_RESOURCE_GROUP="myIdentityResourceGroup"
export IDENTITY_NAME_1="application-identity_1"
az identity create --resource-group ${IDENTITY_RESOURCE_GROUP} --name ${IDENTITY_NAME_1}
export IDENTITY_NAME_2="application-identity_2"
az identity create --resource-group ${IDENTITY_RESOURCE_GROUP} --name ${IDENTITY_NAME_2}
export IDENTITY_CLIENT_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME_1} --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME_1} --query id -otsv)"
export IDENTITY_CLIENT_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME_2} --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME_2} --query id -otsv)"
```

## Assign permissions for the managed identities

The *IDENTITY_CLIENT_ID* managed identity must have Reader permissions in the resource group that contains the virtual machine scale set of your AKS cluster.

```azurecli-interactive
NODE_GROUP=$(az aks show -g myResourceGroup -n myAKSCluster --query nodeResourceGroup -o tsv)
NODES_RESOURCE_ID=$(az group show -n $NODE_GROUP -o tsv --query "id")
az role assignment create --role "Reader" --assignee "$IDENTITY_CLIENT_ID_1" --scope $NODES_RESOURCE_ID
az role assignment create --role "Reader" --assignee "$IDENTITY_CLIENT_ID_2" --scope $NODES_RESOURCE_ID
```

## Create pod identities

Create pod identities for the cluster using `az aks pod-identity add`.

> [!IMPORTANT]
> You must have the appropriate permissions, such as `Owner`, on your subscription to create the identity and role binding.

```azurecli-interactive
export POD_IDENTITY_NAME="my-pod-identity"
export POD_IDENTITY_NAMESPACE="my-app"
az aks pod-identity add --resource-group myResourceGroup --cluster-name myAKSCluster --namespace ${POD_IDENTITY_NAMESPACE}  --name ${POD_IDENTITY_NAME} --identity-resource-id ${IDENTITY_RESOURCE_ID_1} --binding-selector foo
az aks pod-identity add --resource-group myResourceGroup --cluster-name myAKSCluster --namespace ${POD_IDENTITY_NAMESPACE}  --name ${POD_IDENTITY_NAME} --identity-resource-id ${IDENTITY_RESOURCE_ID_2} --binding-selector foo
```

> [!NOTE]
> When you enable pod-managed identity on your AKS cluster, an AzurePodIdentityException named *aks-addon-exception* is added to the *kube-system* namespace. An AzurePodIdentityException allows pods with certain labels to access the Azure Instance Metadata Service (IMDS) endpoint without being intercepted by the node-managed identity (NMI) server. The *aks-addon-exception* allows AKS first-party addons, such as AAD pod-managed identity, to operate without having to manually configure an AzurePodIdentityException. Optionally, you can add, remove, and update an AzurePodIdentityException using `az aks pod-identity exception add`, `az aks pod-identity exception delete`, `az aks pod-identity exception update`, or `kubectl`.

## Run a sample application with multiple identities

For a pod to use AAD pod-managed identity, the pod needs an *aadpodidbinding* label with a value that matches a selector from a *AzureIdentityBinding*. To run a sample application using AAD pod-managed identity, create a `demo.yaml` file with the following contents. Replace *POD_IDENTITY_NAME*, *IDENTITY_CLIENT_ID*, and *IDENTITY_RESOURCE_GROUP* with the values from the previous steps. Replace *SUBSCRIPTION_ID* with your subscription ID.

> [!NOTE]
> In the previous steps, you created the *POD_IDENTITY_NAME*, *IDENTITY_CLIENT_ID*, and *IDENTITY_RESOURCE_GROUP* variables. You can use a command such as `echo` to display the value you set for variables, for example `echo $IDENTITY_NAME`.

```yml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    aadpodidbinding: foo
spec:
  containers:
  - name: demo
    image: mcr.microsoft.com/oss/azure/aad-pod-identity/demo:v1.6.3
    args:
      - --subscriptionid=$SUBSCRIPTION_ID
      - --clientid=$IDENTITY_CLIENT_ID
      - --resourcegroup=$IDENTITY_RESOURCE_GROUP
    env:
      - name: MY_POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: MY_POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: MY_POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
  nodeSelector:
    kubernetes.io/os: linux
```

Notice the pod definition has an *aadpodidbinding* label with a value that matches the name of the pod identity you ran `az aks pod-identity add` in the previous step.

Deploy `demo.yaml` to the same namespace as your pod identity using `kubectl apply`:

```azurecli-interactive
kubectl apply -f demo.yaml --namespace $POD_IDENTITY_NAMESPACE
```

Verify the sample application successfully runs using `kubectl logs`.

```azurecli-interactive
kubectl logs demo --follow --namespace $POD_IDENTITY_NAMESPACE
```

Verify the logs show the a token is successfully acquired and the *GET* operation is successful.
 
```output
...
successfully doARMOperations vm count 0
successfully acquired a token using the MSI, msiEndpoint(http://169.254.169.254/metadata/identity/oauth2/token)
successfully acquired a token, userAssignedID MSI, msiEndpoint(http://169.254.169.254/metadata/identity/oauth2/token) clientID(xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
successfully made GET on instance metadata
...
```
export IDENTITY_CLIENT_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME} --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME} --query id -otsv)"
```

## Clean up

To remove AAD pod-managed identity from your cluster, remove the sample application and the pod identity from the cluster. Then remove the identity.

```azurecli-interactive
kubectl delete pod demo --namespace $POD_IDENTITY_NAMESPACE
az aks pod-identity delete --name ${POD_IDENTITY_NAME} --namespace ${POD_IDENTITY_NAMESPACE} --resource-group myResourceGroup --cluster-name myAKSCluster
az identity delete -g ${IDENTITY_RESOURCE_GROUP} -n ${IDENTITY_NAME}
```

## Next steps

For more information on managed identities, see [Managed identities for Azure resources][az-managed-identities].

<!-- LINKS - external -->
[az-aks-create]: /cli/azure/aks#az_aks_create
[az-aks-get-credentials]: /cli/azure/aks#az_aks_get_credentials
[az-extension-add]: /cli/azure/extension#az_extension_add
[az-extension-update]: /cli/azure/extension#az_extension_update
[az-group-create]: /cli/azure/group#az_group_create
[az-identity-create]: /cli/azure/identity#az_identity_create
[az-managed-identities]: ../active-directory/managed-identities-azure-resources/overview.md
[az-role-assignment-create]: /cli/azure/role/assignment#az_role_assignment_create