### Set Subscription

    az account set --subscription "<subscription_id>"

### Create Resource Group

    az group create -n agilek-rg -l eastus

### Create Serivce Principal

    az ad sp create-for-rbac --name agilek-sp

### Create the AKS Cluster

    az aks create -g agilek-rg -n agilek -k 1.16.4 --service-principal "<app_id>" --client-secret "<client_secret>" --dns-name-prefix "agilek-dns" --enable-addons monitoring --workspace-resource-id "/subscriptions/<subscription_id>/resourcegroups/defaultresourcegroup-eus/providers/microsoft.operationalinsights/workspaces/agilek-logs" --enable-cluster-autoscaler --enable-vmss --location eastus --max-count 7 --min-count 3 --node-count 3 --node-vm-size Standard_B2ms --nodepool-name agilekpool --tags customer=agileways

### Update the cluster to be attached to your ACR

    az aks update -n agilek -g agilek-rg --attach-acr agilewaysreg

_(link https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration)_

### Set up local context for cluster

    az aks get-credentials -g agilek-rg -n agilek

### Add a second nodepool to the cluster, using taints

    az aks nodepool add --cluster-name agilek --name mygpupool --enable-cluster-autoscaler -k 1.16.4 --max-count 7 --min-count 2 --node-count 2 --node-vm-size Standard_B2s --resource-group agilek-rg --node-taints dest=gpu:NoSchedule

_Can also use `kubectl` for tainting (https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#schedule-pods-using-taints-and-tolerations)_

### Create Flux Namespace

    kubectl create ns fluxcd

### Deploy Flux to Cluster via Helm3

    helm3 upgrade -i flux fluxcd/flux --wait --namespace fluxcd --set registry.pollInterval=1m --set git.pollInterval=1m --set git.url=git@ssh.dev.azure.com:v3/david-hoerster/AKS%20Demo%20with%20Cloud%20Native%20Tools/scripts

### Get the public key that Flux uses for accessing DevOps

    fluxctl identity --k8s-fwd-ns fluxcd

### Apply the Helm Operator chart to cluster

    kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/flux-helm-release-crd.yaml

    helm3 upgrade -i helm-operator fluxcd/helm-operator --wait --namespace fluxcd --set git.ssh.secretName=flux-git-deploy --set git.pollInterval=1m --set chartsSyncInterval=1m --set helm.versions=v3

### Install KEDA

1. Add Helm Repo
   - `helm repo add kedacore https://kedacore.github.io/charts`
2. Update Helm Repo
   - `helm repo update`
3. Install Helm Chart
   - `kubectl create namespace keda`
   - `helm install keda kedacore/keda --namespace keda`

### Deploy Linkerd Service Mesh

    linkerd2 check --pre

    linkerd2 install | kubectl apply -f -

    linkerd2 check

### Add Flagger chart

    helm3 repo add flagger https://flagger.app

    kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml

    helm3 upgrade -i flagger flagger/flagger --wait --namespace linkerd --set crd.create=false --set metricsServer=http://linkerd-prometheus:9090 --set meshProvider=linkerd

### Deploy Kong Ingress Controller

    helm3 repo add kong https://charts.konghq.com

    helm3 repo update

_deployed via HelmRelease (kong.yaml and kong-install.yaml)_

#### _Legacy Calls_

    curl -sL https://bit.ly/k4k8s | kubectl create -f -

### Deploy NGINX Ingress Controller

    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/common/ns-and-sa.yaml
    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/rbac/rbac.yaml

    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/common/default-server-secret.yaml
    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/common/nginx-config.yaml
    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/common/custom-resource-definitions.yaml

    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/deployment/nginx-ingress.yaml

    kubectl get pods --namespace=nginx-ingress

    kubectl apply -f https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/service/loadbalancer.yaml

### Deploy Hipster Demo App

_deployed via FluxCD (hipster.yaml)_

#### _Legacy Calls_

    kubectl create namespace hipster

    kubectl apply -f ./release/kubernetes-manifests.yaml -n hipster

### Associate Hipster App with Service Mesh

    cat kubernetes-manifests.yaml | linkerd2 inject - | kubectl apply -n hipster -f -

### Deploy Sealed Secrets to Cluster

commit `sealed-secrets.yaml` to repo

#### create public key

    kubeseal --fetch-cert \
    --controller-namespace=fluxcd \
    --controller-name=sealed-secrets \
    > pub-cert.pem

#### and generate secret...

    kubectl -n prod create secret generic basic-auth \
    --from-literal=user=admin \
    --from-literal=password=admin \
    --dry-run \
    -o json > basic-auth.json

    kubeseal --format=yaml --cert=pub-cert.pem < basic-auth.json > basic-auth.yaml

### Check the Linkerd Dashboard

    linkerd2 dashboard &

### Install Flagger

    kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml

    helm3 upgrade -i flagger flagger/flagger --wait \
    --namespace linkerd \
    --set crd.create=false \
    --set metricsServer=http://linkerd-prometheus:9090 \
    --set meshProvider=linkerd

### Deploy Voting App

    //apply namespace
    kubectl apply -f namespace.yaml

    //deploy service and deployment
    kubectl apply -f deployment.yaml -n voting

    //deploy the ingress entry
    kubectl apply -f ingress.yaml -n voting

### Associate Voting App with Service Mesh

    cat deployment.yaml | linkerd2 inject - | kubectl apply -n voting -f -
