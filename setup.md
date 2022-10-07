1. Prerequisits

   ```bash
   az feature register --namespace "Microsoft.ContainerService" -n "EnablePodIdentityPreview"

   # Keep running until all say "Registered." (This may take up to 20 minutes.)
   az feature list -o table --query "[?name=='Microsoft.ContainerService/EnablePodIdentityPreview'].{Name:name,State:properties.state}"

   az feature register --namespace "Microsoft.ContainerService" -n "KubeletDisk"

   # Keep running until all say "Registered." (This may take up to 20 minutes.)
   az feature list -o table --query "[?name=='Microsoft.ContainerService/KubeletDisk'].{Name:name,State:properties.state}"

   # When all say "Registered" then re-register the AKS resource provider
   az provider register --namespace Microsoft.ContainerService
   ```

1. Get modified repository from github and switch to branch nofw

   ```bash
   GITHUB_ACCOUNT_NAME=ibernato

   git clone https://github.com/${GITHUB_ACCOUNT_NAME}/aks-baseline-regulated.git
   cd aks-baseline-regulated

   git checkout nofw
   ```

1. Prepare certificates for Application Gateway listener (*.scalenesolutions.com) and internal (*.internal.scalenesolutions.com)

   * Listener
   ```bash
   APP_GATEWAY_LISTENER_CERTIFICATE_BASE64=$(cat certificate.pfx | base64 | tr -d '\n')
   ```

   * Internal
   ```bash
   INGRESS_CONTROLLER_CERTIFICATE_BASE64=$(cat internal.crt | base64 | tr -d '\n')
   ```

1. Configure Azure AD

   * Create kubernetes administration group
   ```bash
   AADOBJECTNAME_GROUP_CLUSTERADMIN=cluster-admins-production
   AADOBJECTID_GROUP_CLUSTERADMIN=$(az ad group create --display-name $AADOBJECTNAME_GROUP_CLUSTERADMIN --mail-nickname $AADOBJECTNAME_GROUP_CLUSTERADMIN --description "Principals in this group are cluster admins in the production cluster." --query objectId -o tsv)
   ```

   * Take domain from current user and apply to newly created user
   ```bash
   TENANTDOMAIN_K8SRBAC=$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d '@' -f 2 | sed 's/\"//')
   AADOBJECTNAME_USER_CLUSTERADMIN=production-admin
   # AADOBJECTID_USER_CLUSTERADMIN=$(az ad user create --display-name=${AADOBJECTNAME_USER_CLUSTERADMIN} --user-principal-name ${AADOBJECTNAME_USER_CLUSTERADMIN}@${TENANTDOMAIN_K8SRBAC} --force-change-password-next-login --password ChangeMebu0001a0005AdminChangeMe --query objectId -o tsv)
   ```

   * (dont have privileges to create so I use my account to be in group for admins, that group gives me permissions to get on kubernetes administration)
   ```bash
   AADOBJECTID_USER_CLUSTERADMIN=$(az ad signed-in-user show --query 'objectId' -o tsv | cut -d '@' -f 2 | sed 's/\"//')
   AADOBJECTID_GROUP_CLUSTERADMIN=$(az ad group show --group $AADOBJECTNAME_GROUP_CLUSTERADMIN --query objectId -o tsv)
   ```

   * Add admin user to group
   ```bash
   az ad group member add -g $AADOBJECTID_GROUP_CLUSTERADMIN --member-id $AADOBJECTID_USER_CLUSTERADMIN
   ```

1. Get information on subscription

   ```bash
   TENANTID_AZURERBAC=$(az account show --query tenantId -o tsv)
   NETWORK_WATCHER_RG_REGION=$(az group list --query "[?name=='networkWatcherRG' || name=='NetworkWatcherRG'].location" -o tsv)
   ```

   * Deploy subscription resources
   ```bash
   # [This may take up to six minutes to run.]   
   # az deployment sub create -f subscription.bicep -l australiaeast -p networkWatcherRGRegion="${NETWORK_WATCHER_RG_REGION}"
   # FAILED TO RUN (ALREADY EXISTS)
   # This failed because we already created deployment with name subscription and I changed file to subscription-new to pick changes
   # I dont know if affects existing deployment.

   # [This may take up to six minutes to run.]
   az deployment sub create -f subscription-new.bicep -l australiaeast -p networkWatcherRGRegion="${NETWORK_WATCHER_RG_REGION}"
   ```
   
1. Create the regional network hub.

   ```bash
   # [This takes about eight minutes to run.]
   az deployment group create -g rg-production-networking-hubs -f networking/hub-region.v0.bicep -p location=australiaeast
   ```

1. Create jumpbox

   * Version 0
   ```bash
   RESOURCEID_VNET_HUB=$(az deployment group show -g rg-production-networking-hubs -n hub-region.v0 --query properties.outputs.hubVnetId.value -o tsv)

   # [This takes about one minute to run.]
   az deployment group create -g rg-production-networking-spokes -f networking/spoke-BU0001A0005-00.bicep -p location=australiaeast hubVnetResourceId="${RESOURCEID_VNET_HUB}"
   ```

   * Version 1
   ```bash
   RESOURCEID_SUBNET_AIB=$(az deployment group show -g rg-production-networking-spokes -n spoke-BU0001A0005-00 --query properties.outputs.imageBuilderSubnetResourceId.value -o tsv)

   # [This takes about five minutes to run.]
   az deployment group create -g rg-production-networking-hubs -f networking/hub-region.v1.bicep -p location=australiaeast aksImageBuilderSubnetResourceId="${RESOURCEID_SUBNET_AIB}"
   ```

   * Build RBAC
   ```bash
   # [This takes about one minute to run.]
   az deployment sub create -f jumpbox/createsubscriptionroles.bicep -l australiaeast -n DeployAibRbacRoles
   ```

   * Build image template
   ```bash
   #ROLEID_NETWORKING=4d97b98b-1d4f-4787-a291-c67834d212e7 # Network Contributor -- Only use this if you did not, or could not, create custom roles. This is more permission than necessary.)
   ROLEID_NETWORKING=$(az deployment sub show -n DeployAibRbacRoles --query 'properties.outputs.roleResourceIds.value.customImageBuilderNetworkingRole.guid' -o tsv)
   #ROLEID_IMGDEPLOY=b24988ac-6180-42a0-ab88-20f7382dd24c  # Contributor -- only use this if you did not, or could not, create custom roles. This is more permission than necessary.)
   ROLEID_IMGDEPLOY=$(az deployment sub show -n DeployAibRbacRoles --query 'properties.outputs.roleResourceIds.value.customImageBuilderImageCreationRole.guid' -o tsv)

   # [This takes about one minute to run.]
   az deployment group create -g rg-production -f jumpbox/azuredeploy.bicep -p buildInSubnetResourceId=${RESOURCEID_SUBNET_AIB} location=australiaeast imageBuilderNetworkingRoleGuid="${ROLEID_NETWORKING}" imageBuilderImageCreationRoleGuid="${ROLEID_IMGDEPLOY}" -n CreateJumpBoxImageTemplate
   ```

   * Build image
   ```bash
   IMAGE_TEMPLATE_NAME=$(az deployment group show -g rg-production -n CreateJumpBoxImageTemplate --query 'properties.outputs.imageTemplateName.value' -o tsv)

   # [This takes about >> 30 minutes << to run.]
   az image builder run -n $IMAGE_TEMPLATE_NAME -g rg-production
   ```

   * Create user for jumpbox, password M#^xJPePu&^2C6kg      
   ```bash
   ssh-keygen -t rsa -b 4096 -f opsuser.key
   cat opsuser.key.pub
   ```

1. Deploy cluster spoke

   ```bash
   RESOURCEID_VNET_HUB=$(az deployment group show -g rg-production-networking-hubs -n hub-region.v0 --query properties.outputs.hubVnetId.value -o tsv)

   # [This takes about five minutes to run.]
   az deployment group create -g rg-production-networking-spokes -f networking/spoke-BU0001A0005-01.bicep -p location=australiaeast hubVnetResourceId="${RESOURCEID_VNET_HUB}"
   ```

   * Update region
   ```bash
   RESOURCEID_SUBNET_AIB=$(az deployment group show -g rg-production-networking-spokes -n spoke-BU0001A0005-00 --query properties.outputs.imageBuilderSubnetResourceId.value -o tsv)
   RESOURCEID_SUBNET_NODEPOOLS="['$(az deployment group show -g rg-production-networking-spokes -n spoke-BU0001A0005-01 --query "properties.outputs.nodepoolSubnetResourceIds.value | join ('\',\'',@)" -o tsv)']"
   RESOURCEID_SUBNET_JUMPBOX=$(az deployment group show -g rg-production-networking-spokes -n spoke-BU0001A0005-01 --query properties.outputs.jumpboxSubnetResourceId.value -o tsv)

   # [This takes about seven minutes to run.]
   az deployment group create -g rg-production-networking-hubs -f networking/hub-region.v2.bicep -p location=australiaeast aksImageBuilderSubnetResourceId="${RESOURCEID_SUBNET_AIB}" nodepoolSubnetResourceIds="${RESOURCEID_SUBNET_NODEPOOLS}" aksJumpboxSubnetResourceId="${RESOURCEID_SUBNET_JUMPBOX}"
   ```

1. Deploy kubernetes cluster

   * Get information
   ```bash
   RESOURCEID_VNET_CLUSTERSPOKE=$(az deployment group show -g rg-production-networking-spokes -n spoke-BU0001A0005-01 --query properties.outputs.clusterVnetResourceId.value -o tsv)
   
   # If you used a pre-existing image and not the one built by this walk through, replace the command below with the resource id of that image.
   RESOURCEID_IMAGE_JUMPBOX=$(az deployment group show -g rg-production -n CreateJumpBoxImageTemplate --query 'properties.outputs.distributedImageResourceId.value' -o tsv)

   CLOUDINIT_BASE64=$(base64 jumpBoxCloudInit.yml | tr -d '\n')
   ```

   * Deploy cluster
   ```bash
   # [This takes about 20 minutes to run.]
   az deployment group create -g rg-production -f cluster-stamp.json -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE} clusterAdminAadGroupObjectId=${AADOBJECTID_GROUP_CLUSTERADMIN} k8sControlPlaneAuthorizationTenantId=${TENANTID_K8SRBAC} appGatewayListenerCertificate=${APP_GATEWAY_LISTENER_CERTIFICATE_BASE64} aksIngressControllerCertificate=${INGRESS_CONTROLLER_CERTIFICATE_BASE64} jumpBoxImageResourceId=${RESOURCEID_IMAGE_JUMPBOX} jumpBoxCloudInitAsBase64=${CLOUDINIT_BASE64}
   ```

   * Redeploy changes (managed identity)
   ```bash
   # [This takes about five minutes to run.]
   az deployment group create -g rg-production -f cluster-stamp.v2.json -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE} clusterAdminAadGroupObjectId=${AADOBJECTID_GROUP_CLUSTERADMIN} k8sControlPlaneAuthorizationTenantId=${TENANTID_K8SRBAC} appGatewayListenerCertificate=${APP_GATEWAY_LISTENER_CERTIFICATE_BASE64} aksIngressControllerCertificate=${AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64} jumpBoxImageResourceId=${RESOURCEID_IMAGE_JUMPBOX} jumpBoxCloudInitAsBase64=${CLOUDINIT_BASE64}
   ```

1. Add certificate to keyvault

   ```bash
   KEYVAULT_NAME=$(az deployment group show --resource-group rg-production -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)
   az keyvault set-policy --certificate-permissions import --upn $(az account show --query user.name -o tsv) -n $KEYVAULT_NAME
   
   az keyvault certificate import -f ingress-internal-aks-ingress-contoso-com-tls.pem -n ingress-internal-aks-ingress-contoso-com-tls --vault-name $KEYVAULT_NAME
   ```

1. Get images to Container Registry

   * Put to quarantine
   ```bash
   ACR_NAME_QUARANTINE=$(az deployment group show -g rg-production -n cluster-stamp --query properties.outputs.quarantineContainerRegistryName.value -o tsv)
   
   az acr import --source ghcr.io/fluxcd/kustomize-controller:v0.8.1 -t quarantine/fluxcd/kustomize-controller:v0.8.1 -n $ACR_NAME_QUARANTINE && \
   az acr import --source ghcr.io/fluxcd/source-controller:v0.8.1 -t quarantine/fluxcd/source-controller:v0.8.1 -n $ACR_NAME_QUARANTINE       && \
   az acr import --source docker.io/falcosecurity/falco:0.29.1 -t quarantine/falcosecurity/falco:0.29.1 -n $ACR_NAME_QUARANTINE               && \
   az acr import --source docker.io/library/busybox:1.33.0 -t quarantine/library/busybox:1.33.0 -n $ACR_NAME_QUARANTINE                       && \
   az acr import --source docker.io/weaveworks/kured:1.9.2 -t quarantine/weaveworks/kured:1.9.2 -n $ACR_NAME_QUARANTINE                       && \
   az acr import --source k8s.gcr.io/ingress-nginx/controller:v1.1.2 -t quarantine/ingress-nginx/controller:v1.1.2 -n $ACR_NAME_QUARANTINE    && \
   az acr import --source k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1 -t quarantine/jettech/kube-webhook-certgen:v1.1.1 -n $ACR_NAME_QUARANTINE
   ```

   * Move to live
   ```bash
   ACR_NAME=$(az deployment group show -g rg-production -n cluster-stamp --query properties.outputs.containerRegistryName.value -o tsv)
   
   az acr import --source quarantine/fluxcd/kustomize-controller:v0.8.1 -r $ACR_NAME_QUARANTINE -t live/fluxcd/kustomize-controller:v0.8.1 -n $ACR_NAME && \
   az acr import --source quarantine/fluxcd/source-controller:v0.8.1 -r $ACR_NAME_QUARANTINE -t live/fluxcd/source-controller:v0.8.1 -n $ACR_NAME       && \
   az acr import --source quarantine/falcosecurity/falco:0.29.1 -r $ACR_NAME_QUARANTINE -t live/falcosecurity/falco:0.29.1 -n $ACR_NAME                 && \
   az acr import --source quarantine/library/busybox:1.33.0 -r $ACR_NAME_QUARANTINE -t live/library/busybox:1.33.0 -n $ACR_NAME                         && \
   az acr import --source quarantine/weaveworks/kured:1.9.2 -r $ACR_NAME_QUARANTINE -t live/weaveworks/kured:1.9.2 -n $ACR_NAME                         && \
   az acr import --source quarantine/ingress-nginx/controller:v1.1.2 -r $ACR_NAME_QUARANTINE -t live/ingress-nginx/controller:v1.1.2 -n $ACR_NAME       && \
   az acr import --source quarantine/jettech/kube-webhook-certgen:v1.1.1 -r $ACR_NAME_QUARANTINE -t live/jettech/kube-webhook-certgen:v1.1.1 -n $ACR_NAME
   ```








