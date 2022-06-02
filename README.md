# crossplane-getting-started

Repository containing my notes on how to get started with Crossplane

## Why Crossplane

Crossplane brings the Kubernetes way of managing infrastructure in grasp for any kind of resource.

## Install

Pre-requisites:

- Kind
- kubectl
- helm

Create a test Kind cluster named `crossplane-test`.

```bash
kind create cluster --image kindest/node:v1.23.0 --wait 5m --name crossplane-test
```

Install crossplane core components using Helm.

```bash
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

Check that the installation succeeded.

```bash
helm list -n crossplane-system

kubectl get all -n crossplane-system
```

Install the Crossplane CLI, it is an extension for `kubectl`

```bash
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh

# Move the crossplane kubectl extension to bin
sudo mv kubectl-crossplane /usr/local/bin

# verify that it is installed
kubectl crossplane --help
```

## Configure Azure

First, let's install the crossplane Azure configuration package.

Use the below K8s manifest to instruct crossplane to use Azure provider. This pulls in the required CRDs and Controllers on the cluster.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure
spec:
  package: "crossplane/provider-azure:master"
```

Create SPI and configure it, this is the service principal that the Azure provider controller will use to spin up resources.

For this article, I kep the API permissions assigned very simple to the SPI e.g. Owner role on the subscription. However, if you're looking to provision.

**Note** - Using `--sdk-auth` is going to be deprecated in future versions of Az CLI. Az CLI is undergoing some changes with using Microsoft Graph API. For this demo, using Az CLI v2.35.0.

```bash
az ad sp create-for-rbac --sdk-auth --role Owner --scopes="/subscriptions/ef95af99-ec71-4ee6-b018-8c4d3d0ebf73" -n "crossplane-sp-rbac" > "creds.json"
```

Create a Kubernetes secret to hold the credentials used by the Azure provider.

```bash
kubectl create secret generic azure-creds -n crossplane-system --from-file=creds=./creds.json
```

Instruct Azure crossplane provider to use these secrets created by using the below mainfest.

```yaml
apiVersion: azure.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-creds
      key: creds
```

Once we have the provider and the provider config in place, it is time to spin up resources in Azure using Kubernetes manifests powered by Crossplane.

```bash
kubectl apply -f az-provider.yaml
```

Now wait for the provider to be up and ready, you can watch the status of the provider by running.

```bash
kubectl get provider --watch
```

## Provision resources

In a very simple scenario, we can now get started with using Azure crossplane provider to spin up resources.

Let's deploy a resource group in Azure using a Kubernetes manifest.

File `test-rg.yaml` content.

```yaml
apiVersion: azure.crossplane.io/v1alpha3
kind: ResourceGroup
metadata:
  name: test-rg-crossplane
spec:
  location: eastus
```

Apply manifest.

```bash
kubectl apply -f test-rg.yaml
```

Once the resource group is deployed, let's add a storage account.

File `test-storage.yaml` content.

Check the status of the resource group by using below:

```bash
kubectl get ResourceGroup --watch
```

```yaml
apiVersion: storage.azure.crossplane.io/v1alpha3
kind: Account
metadata:
  name: teststoragecrossplane999
spec:
  resourceGroupName: test-rg-crossplane
  storageAccountSpec:
    kind: Storage
    location: eastus
    sku:
      name: Standard_LRS
  writeConnectionSecretToRef:
    namespace: default
    name: storageaccount-connection-secret
```

Apply manifest.

```bash
kubectl apply -f test-storage.yaml
```

See the status of the resources provisioned.

```bash
kubectl get resourcegroup # for RG
kubectl get account # for storage account
kubectl get secret -n default # check if the writeConnectionSecretToRef worked and created a secret
```

Wait until the state is `Synced` on the resources, they will now reflect in Azure.

One benefit of using Kubernetes declarative APIs is that it takes care of the configuration drift. Try deleting the storage account from portal and see them get re-created.

## More Crossplane concepts

Crossplane has below two concepts that allow us to extend and build on top of the default resources being exposed for a Cloud provider.

- `CompositeResourceDefinition` (XRD) this is a blueprint of a composite resource. It can optionally offer a claim e.g. XRC.
- `Composition` this is the grouping of one or more composite resource. This defines how the composite resource(s) are configured.
