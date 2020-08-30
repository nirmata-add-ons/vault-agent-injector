# Nirmata Vault Agent Injector Add-On

This repository contains YAML files and instructions for configuring the [Hashicorp Vault Agent Injector](https://www.vaultproject.io/docs/platform/k8s/injector) as a Nirmata add-on service.

## Usage

To configure the add-on, follow these steps:
1. Clone this repository or add its contents to your own private Git repository. You can also change the AGENT_INJECT_VAULT_ADDR to contain the address of your Vault service (this can also be set in Nirmata.)
2. Create a Nirmata catalog application with a Git upstream and select your repository. The name of the application must be set to **vault-agent-injector** so Nirmata can auto-configure the required settings.
3. Edit the catalog application and select an add-on category.
4. Update cluster types, or create new ones, and select the add-on.
5. Create clusters using the cluster type.

**Note:** if you select the stable channel, make sure you have a release available to deploy to the cluster. 

## Vault Initial Setup

The Vault Agent Injector requires the Vault Kubernetes authentication method which uses Kubernetes RBAC and security to authenticate workloads. This auth method requires each Kubernetes cluster to have its own path in Vault. 

Once a new cluster is provisioned, Nirmata will automatically create the required configuration for Vault. Simply click on the add-on entry and follow the configuration steps to complete Vault setup.

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



