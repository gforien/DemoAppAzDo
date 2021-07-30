# Pipeline de CI/CD Azure DevOps ACR + AKS

## Table des matières
1. [**Créer un compte et installer Azure CLI**](#1-créer-un-compte-et-installer-azure-cli)
1. [**Créer un environnement ACR + AKS**](#2-créer-un-environnement-acr--aks)
1. [**Créer un environnement ACR + AKS avec Azure CLI depuis Docker**](#3-créer-un-environnement-acr--aks-avec-azure-cli-depuis-docker)
1. [**Créer le pipeline**](#4-créer-le-pipeline)

<ins>**Pour plus tard:**</ins> [CI/CD on GKE](https://cloud.google.com/architecture/creating-cicd-pipeline-vsts-kubernetes-engine) et [CI/CD on EKS](https://github.com/aws-samples/amazon-eks-cicd-codebuild)

<ins>**Les samples sur GitHub:**</ins> [Azure Samples](https://github.com/Azure-Samples), [AWS Samples](https://github.com/aws-samples), et [GCP Samples](https://github.com/GoogleCloudPlatform?q=samples)

<ins>**Le EKS workshop:**</ins> [eksworkshop.com](https://www.eksworkshop.com/) et le [respository github](https://github.com/aws-samples/aws-workshop-for-kubernetes) associé

---
## 1. Créer un compte et installer Azure CLI
[📚 **Source**](https://devblogs.microsoft.com/devops/using-azure-devops-from-the-command-line/)


On installe [Azure CLI](https://docs.microsoft.com/fr-fr/cli/azure/install-azure-cli-windows)
et il y a aussi [Azure CLI Classic](https://docs.microsoft.com/fr-fr/cli/azure/install-classic-cli)

il y a dev.azure.com sur lequel on peut créer une organisation facilement
mais pour créer un compte azure c'est portal.azure.com

```sh
az login
    {
    "cloudName": "AzureCloud",
    "homeTenantId": "xxxxxxxxxxxxxxxxx",
    "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Essai gratuit",
    "state": "Enabled",
    "tenantId": "xxxxxxxxxxxxxxxxx",
    "user": {
      "name": "monadresse@outlook.com",
      "type": "user"
    }
    }

az extension list-available
az extension add --name azure-devops
az extension list
    {
      "experimental": false,
      "extensionType": "whl",
      "name": "azure-devops",
      "path": "C:\\Users\\moi\\.azure\\cliextensions\\azure-devops",
      "preview": false,
      "version": "0.18.0"
    }

az devops configure --defaults organization=https://dev.azure.com/mon-orga project=mypipe

az pipelines build list -o table
```

## 2. Créer un environnement ACR + AKS

[📚 **Source**](https://github.com/thomasrannou/DemoAppAzDo/blob/main/createEnvACRAKS.ps1) (avec Azure CLI)<br>
[📚 **Source n°2**](https://thomasrannou.azurewebsites.net/2020/12/16/automatiser-le-deplo-dun-cluster-aks-avec-terraform-et-azure-devops/) (en GUI sur la plateforme Azure)
```sh

az acr create --name $registry --resource-group $aksrg --sku basic
# Resource provider 'Microsoft.ContainerRegistry' used by this operation is not registered. We are registering for you.

az aks create --name $aks --resource-group $aksrg --attach-acr $registryId --generate-ssh-keys --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --node-count 2 --zones 1
# SSH key files 'id_rsa' and 'id_rsa.pub' have been generated under ~/.ssh to allow SSH access to the VM. If using machines without permanent storage like Azure Cloud Shell without an attached file share, back up your keys to a safe location

# Resource provider 'Microsoft.ContainerService' used by this operation is not registered. We are registering for you.

# → ça prend trop de temps
```

On troubleshoot 🔥

#### [Mettre à jour Azure CLI](https://docs.microsoft.com/fr-fr/cli/azure/update-azure-cli)
```sh
az upgrade
```

#### Vérifier son [account subscription](https://docs.microsoft.com/fr-fr/cli/azure/account/subscription)
```sh
az account set --subscription xxxxxxxxxxxxxxxxxxxxxxxxxx
az account subscription list
# xxxxxxxxxxxxxxxxxxxx
```

#### Vérifier son contexte Kubernetes
```sh
k config view 
    # bla bla bla...
k config current-context
    #> clusterAKS
k config use-context docker-desktop
    #> Switched to context "docker-desktop".
k describe service
    #> Name:              kubernetes
    #> Namespace:         default
    #> Labels:            component=apiserver
    #>                    provider=kubernetes
    #> Annotations:       <none>
    #> Selector:          <none>
    #> Type:              ClusterIP
    #> IP:                10.96.0.1
    #> Port:              https  443/TCP
    #> TargetPort:        6443/TCP
    #> Endpoints:         192.168.65.3:6443
    #> Session Affinity:  None
    #> Events:            <none>
```

#### Vérifier que son client PS/Kubernetes a les permissions (pas réussi)
```sh
kubectl describe service my-service
    #> Code="LinkedAuthorizationFailed" Message="The client 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' with object id 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' has permission to perform action 'Microsoft.Network/loadBalancers/write' on scope '/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.Network/loadBalancers/kubernetes-internal'; however, it does not have permission to perform action 'Microsoft.Network/virtualNetworks/subnets/join/action' on the linked scope(s) '/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>' or the linked scope(s) are invalid.

az role assignment create `
    --role Owner `
    --assignee xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

#### Éventuellement [passer à WSL2 ?](https://dev.to/wrightdotclick/20x-faster-speeds-by-updating-to-the-new-wsl-2-a-user-s-installation-guide-57n4)
```sh
wsl -l -v
    #>   NAME            STATE           VERSION
    #> * Ubuntu-18.04    Running         1
```

#### Eventuellement tester la [Azure Voting App](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)

#### Pour l'autocomplétion
Azure CLI est fait pour être utilisé sous bash, il n'implémente pas l'autocomplétion pour Powershell (voir [open issue](https://github.com/Azure/azure-cli/issues/2324)). Mais c'est parce qu'il y a déjà un module Powershell / une image Docker [Azure Powershell](https://hub.docker.com/_/microsoft-azure-powershell) qui implémentent l'autocomplétion pour Powershell, et qui est powershell-like. Voir aussi ["Choosing the right CLI tool"](https://docs.microsoft.com/en-us/azure/developer/azure-cli/choose-the-right-azure-command-line-tool)

#### Lancer Azure CLI depuis Docker
```sh
docker run -it mcr.microsoft.com/azure-cli
```

Ça marche ✅ on continue ➡

## 3. Créer un environnement ACR + AKS avec Azure CLI depuis Docker
```sh
az login
az account list
az account subscription list

location="francecentral"
aksrg="rg-aks"
aks="clusterAKS"
registry="registryACRDemo"

# Création du groupe de ressource
az group create --name $aksrg --location $location
az group list
    #> OK

# Création de la registry
az acr create --name $registry --resource-group $aksrg --sku basic
az acr list
    #> OK

# Id de la registry
registryId=$(az acr show --name $registry --resource-group $aksrg --query "id" --output tsv)

# Création du cluster AKS avec zone de disponibilité
az aks create --name $aks --resource-group $aksrg --attach-acr $registryId --generate-ssh-keys --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --node-count 2 --zones 1
    #> on attend 4 min...
az aks list
    #> OK

# Récupération de l'id du cluster AKS
aks_resourceId=$(az aks show -n $aks -g $aksrg --query id -o tsv)

# En attente du déploiement
az resource wait --exists --ids $aks_resourceId

# si kubectl n'est pas installé
az aks install-cli
echo -e "\nalias k=kubectl" >> ~/.bashrc && source ~/.bashrc

# Connexion au cluster AKS
az aks get-credentials --resource-group $aksrg --name $aks  --overwrite-existing

# Récupération des infos du cluster
k cluster-info
    #> Kubernetes control plane is running at https://clusteraks-rg-aks-7....
    #> CoreDNS is running at https://clusteraks-rg-aks-77d...
    #> Metrics-server is running at https://clusteraks-rg-aks-77d...
```

On a donc un environnement Azure avec un RG contenant un ACR et un cluster AKS (2 nodes)

## 4. Créer le pipeline

[📚 **Source**](https://thomasrannou.azurewebsites.net/2021/01/08/configurer-un-pipeline-ci-cd-azure-devops-pour-deployer-dans-aks/)

1. on fork son [projet github](https://github.com/thomasrannou/DemoAppAzDo)
1. on crée un Pipeline, il y a donc son `azure-pipelines.yml` mais on `Save` uniquement

    Avant de l’exécuter, j’ai besoin d’ajouter 2 services connexion à mon projet Azure DevOps, une connexion à mon CRegistry Azure et une seconde à mon cluster AKS.

1. on va dans `Project Settings`, `Service connections`, on crée une nouvelle connection de type `Docker Registry`, enfin on sélectionne `Azure Connection Registry` pour le connecter directement à notre **registryACRDemo**

**Problème**: impossible de sélectionner le registry<br>
➡ Il faut un access level de type `Owner` dans le permissions de la subscription comme expliqué [ici](https://stackoverflow.com/questions/63050179/azure-devops-service-connection-to-azure).<br>
➡ sur le portail Azure on va dans la subscription `Essai gratuit`, puis `Access control (IAM)` et on clique sur `Add role assignment` pour m'ajouter le rôle Owner.<br>
On continue ✅

1. Même problème avec la connection Kubernetes. Je m'assigne un role `Owner` sur **clusterAKS** et je désactive les protections Brave pour que la popup puisse s'ouvrir ✅

1. On retourne sur le pipeline pour ajouter des variable globales
    - `containerRegistry = registryacrdemon.azurecr.io`
    - `imageRepository = applicationnetcore`

1. Il manque le package [**replacetokens**](https://marketplace.visualstudio.com/items?itemName=qetza.replacetokens)

1. Impossible de build car impossible de lancer des projets en parallèle. Lu le [billet de blog de Microsoft](https://devblogs.microsoft.com/devops/change-in-azure-pipelines-grant-for-private-projects/) à ce sujet, et rempli ma demande.
    - [Solution sur r/azuredevops](https://www.reddit.com/r/azuredevops/comments/mvif7g/unable_to_run_pipeline/) pour lancer un self hosted agent
    - Je suis [ces instructions](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops) qui me font seulement créer un PAT
    - Je lance [l'image Docker d'un Azure Pipelines Agent](https://hub.docker.com/_/microsoft-azure-pipelines-vsts-agent)
    - en fait l'image est deprecated, et plus mise à jour depuis 5 ans comme écrit sur [leur Github](https://github.com/microsoft/vsts-agent-docker)
    - Je trouve les [nouvelles instructions](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker) pour lancer l'image Docker d'un agent en précisant les variables d'environnement, sur un container Windows avec HyperV
    - De plus, l'agent doit pouvoir accéder à Docker (pseudo docker-in-docker), et c'est logique car c'est un agent de CI. Il faut donc passer au container Linux et binder la socker. Voir les réponses stackoverflow [ici](https://stackoverflow.com/questions/36765138) et [ici](https://stackoverflow.com/questions/54062327) pour binder la socket sous Windows et une [issue github](https://github.com/microsoft/azure-pipelines-tasks/issues/12140) où le mec choisit plutôt de faire un vrai docker-in-docker

1. Donc, j'ai mon propre [dockeragent](https://github.com/gforien/azure-dockeragent) capable de faire un pseudo docker-in-docker. Il est seul dans son pool `mydockerpool`, et j'ai modifié `azure-pipelines.yaml` pour que les deux jobs `Build` et `Deploy` se fassent dans ce pool. ➡ Plus de build en parallèle, et plus d'erreur ✅

1. Je lance un build + deploy en exécutant le pipeline

1. Si je modifie `index.cshtml` directement sur Github et que je fais une PR (attention à ne pas l'accepter trop vite), le pipeline se relance automatiquement
    - checkout latest commit
    - build
    - push image to ACR
    - replace tokens in yaml files
    - create [pull secret](https://kubernetes.io/fr/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials)
    - deploy via [rolling update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)

1. Déployé à [**http://40.66.61.211/**](http://40.66.61.211/)
