# Nirmata Vault Agent Injector Add-On

This repository contains YAML files and instructions for configuring the [Hashicorp Vault Agent Injector](https://www.vaultproject.io/docs/platform/k8s/injector) as a Nirmata add-on service.

## Usage

To configure the Vault Injector add-on, follow these steps:
1. Clone this repository or add its contents to your own private Git repository. 
3. Create a Nirmata catalog application with a Git upstream and select the Vault Agent Injector repository. You can optionally select the kustomization (see below.)
4. Edit the catalog application and select an add-on category (e.g. security). This is required to select the application as a add-on.
5. Create a Vault Credentials (Settings -> Integrations -> Vault). This requires a Vault address and a token for Nirmata. The token must be configured with an [access policy](#vault-configuration-for-nirmata-access) that allows Nirmata to configure Kubernetes authentication paths. See details below in the **Vault Configuration for Nirmata Access** section. 
5. Update a Cluster Type, or create a new one, and select the Vault Agent Injector add-on application in the "Add-Ons" section.
6. In the Cluster Type setting also configure the Vault Auth Settings section. Select the Vault Credentials. For the "Kubernetes Authentication Path" you can use `$(cluster.name)` to create a path with the cluster name. For example, `kubernetes/$(cluster.name)`. Also, enter the Vault Injector add-on application name. Nirmata will now watch for that application to be deployed and automatically configure the Vault Kubernetes Authentication path for new clusters with the default roles roles.
6. Create clusters using the cluster type.
7. Verify that the Kubernetes Authentication Path is created in Vault and has the correct roles.
8. Deploy applications that use the standard [Vault Agent Injector annotations](https://www.vaultproject.io/docs/platform/k8s/injector/annotations). 

**Note:** if you select the stable channel, make sure you have a release available in that channel to deploy to the cluster. 

## Customizing the Vault Agent Injector YAML

You can use [kustomize](https://kubernetes-sigs.github.io/kustomize/) to change configuration settings for the Vault Agent Injector. A sample [kustomization.yaml](kustomization.yaml) file with patches to update the `AGENT_INJECT_VAULT_ADDR` and `AGENT_INJECT_VAULT_AUTH_PATH` is provided in the repository.

You can clone the `nirmata-add-ons/vault-agent-injector` repository and run `kustomize` as follows:

```
git clone github.com/nirmata-add-ons/vault-agent-injector
cd vault-agent-injector
kubectl kustomize ./
```

Or, to simply run the kustomization without cloning use: 

```
kubectl kustomize github.com/nirmata-add-ons/vault-agent-injector
```

## Vault Configuration for Nirmata Access

To manage a cluster's Kubernetes Auth path in Vault, Nirmata needs access permissions to enable and configure Kubernetes Auth paths. 

Here is an sample HCL policy that provides Nirmata with permissions to create and configure Kubernetes Auth methods for clusters and limits which policies can be used:

```
# Policy for Nirmata to manage Kubernetes Authentication methods

# allow creation of Kubernetes auth methods in a specific path
path "sys/auth/nirmata/*" {
  capabilities = [ "create", "read", "update", "delete", "sudo" ]
  allowed_parameters = {
    "type" = ["kubernetes"]    
    "*"    = []
  }
}

# allow configuring Kubernetes auth methods in a specific path
path "auth/nirmata/+/config" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# allow adding roles to a specific auth path with specified policies
path "auth/nirmata/+/role/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
  allowed_parameters = {
    # allows roles to use app-policy1, app-policy2, or both
    "policies" = [ ["app-policy1"], ["app-policy2"], ["app-policy1", "app-policy2"] ]
    "*"    = []
  }
  denied_parameters = {
    "token_policies" = []
    "token_no_default_policy" = ["true"]
  }
}


```

This policy must be mapped to the AppRole or access token used by Nirmata to access Vault.

## Workload Authentication

For each application that requests secrets from Vault, you will need to configure a role in Vault and map the role to a policy that allows access to one or more paths that contain the secrets for the application. Check the [Vault documentation](https://www.vaultproject.io/docs/auth/kubernetes.html#configuration) for additional details.

## Accessing Secrets

You can now run workloads with annotations that will instruct the injector to add an init container, a side-car container, and secrets to Pods. The complete list of annotations is available in the [Vault documentation](https://www.vaultproject.io/docs/platform/k8s/injector/annotations).

Here is an example of a Pod that access a secret. The application service account which maps to the Vault role should already exist, or must be created first:

```yaml
kind: "Pod"
apiVersion: "v1"
metadata:
  name: nginx
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "nginx-app"
    vault.hashicorp.com/agent-inject-secret-credentials.txt: "secret/data/nginx/config"
spec:
  serviceAccountName: nginx-app
  containers:
  - name: "nginx"
    image: "nginx:latest"
    ports:
    - containerPort: 8080
      protocol: "TCP"
 
```



