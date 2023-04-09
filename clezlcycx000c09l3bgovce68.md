---
title: "Installing HashiCorp Vault + ExternalSecrets Operator on Kubernetes: the easy way"
seoDescription: "Want to play with Vault and ExternalSecrets, but don't want to spend a day setting them up? Here is the perfect repo for you."
datePublished: Wed Mar 08 2023 11:23:11 GMT+0000 (Coordinated Universal Time)
cuid: clezlcycx000c09l3bgovce68
slug: installing-hashicorp-vault-externalsecrets-operator-on-kubernetes-the-easy-way
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678272116504/048e004d-7e0b-44d0-9841-9f14b5ee174d.png
tags: kubernetes, k3d, hashicorp-vault

---

I recently had to test the new [ExternalSecrets operator](https://external-secrets.io/main) and its capabilities when using [HashiCorp Vault](http://vaultproject.io/) as a backend. I spent some time figuring out how to install them on a local [K3D](https://k3d.io) cluster and wanted to share it so you won't have to.

⮕ ✨✨ [https://github.com/derlin/externalsecrets-with-hashicorp-vault-kubernetes-easy-install](https://github.com/derlin/externalsecrets-with-hashicorp-vault-kubernetes-easy-install) ✨✨

**IMPORTANT**: this is for test purposes only, it is not suitable for production!

---

* [About Vault and ExternalSecrets](#heading-about-vault-and-externalsecrets)
    
* [Installing Vault and ExternalSecrets](#heading-installing-vault-and-externalsecrets)
    
    * [Prerequisites](#heading-prerequisites)
        
    * [Procedure](#heading-procedure)
        
* [Accessing the vault](#heading-accessing-the-vault)
    

---

## About Vault and ExternalSecrets

Kubernetes' `Secrets` resources are a way to store sensitive information. Those Secret resources may be created directly (YAML files), from a Helm Chart, from Kustomize, etc. If you follow a *gitops approach* (you should!) those YAML files, Helm Charts, etc. will live in a git repo somewhere. But you don't want to commit sensitive information in git repos, so how to proceed?

A good approach instead is to use a secret management system (there are plenty to choose from: [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [HashiCorp Vault](https://www.vaultproject.io/), [Google Secrets Manager](https://cloud.google.com/secret-manager), [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), [IBM Cloud Secrets Manager](https://www.ibm.com/cloud/secrets-manager), etc.), and to have a way to retrieve those secrets dynamically from your Kubernetes cluster. This is where ExternalSecrets shines.

ExternalSecrets is a cluster-wide operator that you install once. Then, instead of creating a `Secret` directly, you create an `ExternalSecret` (a custom resource) that defines what secrets to retrieve, and from which backend. The operator then creates the `Secret` for you.

Backends are configured by creating `SecretStore` or `ClusterSecretStore` resources, which hold the connection information to a given secret management system. Each `ExternalSecret` must reference one of those secret stores, so the operator knows from which backend it should retrieve secrets.

The documentation at [https://external-secrets.io/main/](https://external-secrets.io/main/) is quite good, so I will stop here.

## Installing Vault and ExternalSecrets

To simplify the installation, I use helmfile, which uses helm under the hood.

### Prerequisites

Hard requirements

* [helm](https://helm.sh) (`brew install helm`)
    
* [helmfile](https://helmfile.readthedocs.io/) (`brew install helmfile`)
    

Soft requirements:

* The [helm diff plugin](https://github.com/databus23/helm-diff) (`helm plugin install https://github.com/databus23/helm-diff`). This is necessary if you plan to use `helmfile apply` and `helmfile diff`
    
* [k3d](https://k3d.io/) to be able to spawn a local Kubernetes cluster on Docker (`brew install k3d`)
    

### Procedure

First, clone the following repo: [https://github.com/derlin/externalsecrets-with-hashicorp-vault-kubernetes-easy-install](https://github.com/derlin/externalsecrets-with-hashicorp-vault-kubernetes-easy-install).

Start a k3d cluster:

```bash
k3d cluster create test --api-port 6550 -p "80:80@loadbalancer"
```

Install Vault, the ExternalSecret operator, and a `ClusterSecretStore` by running the following at the root of the repository:

```bash
helmfile sync
```

The above command will:

1. Launch a Vault instance in the `vault` namespace, and configure it with a token `root` ,
    
2. Install the ExternalSecrets operator in the `es` namespace,
    
3. Create a `ClusterSecretStore` resource named `vault-backend` in the `default` namespace, which connects to the Vault. You can reference it in an ExternalSecret resource using:
    
    ```yaml
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    # ...
    spec:
      secretStoreRef:
        name: vault-backend
        kind: ClusterSecretStore
    # ...
    ```
    
4. Create a secret in the vault under the path `secret/foo` with one property, `hello`.
    

Done! Now, you can test the operator by creating an `ExternalSecret` resource and wait for the `Secret` `test` to be created:

```bash
kubectl apply -f extsecret-example.yaml
```

## Accessing the vault

The setting above automatically creates the `secret/foo` for you. To **access the vault interface** and add more secrets, create a port forward to access the vault:

```bash
kubectl port-forward -n vault vault-0 8200
```

You can now go to http://localhost:8200 and log in with the default token `root`.

To access the vault **using the command line** (and assuming the port-forwarding is still on):

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root

vault kv get secret/foo
```

You can also set secrets programmatically using `kubectl exec`:

```bash
kubectl exec vault-0 -n vault -- vault kv put secret/foo app-secret-key=123
```