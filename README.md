# vault-k8s

`vault-k8s` is an attempt to ease the setup of a Vault cluster running on Kubernetes. The aim of this repository is to facilitate the deployment of an high-availability 5 nodes Vault cluster using the Raft integrated storage. A tiny script `vk` whitin this repo will help to achieve that.

The idea is to have an ad-hoc, reproducible, configurable, extendable way of deploying a Vault cluster in a dedicated cluster. It also helps to deploy the agent on the Kubernetes cluster(s) of your choice, giving you the freedom to fetch secrets from your Vault and inject them before scheduling pods.

You will find a [section](https://github.com/TommyStarK/vault-k8s#production-readiness) gathering the different links I found useful to decide how to properly deploy this stack to be used in production and meet my needs/constraints. It covers things like capacity planning, pod resources, data persistence, using Ingress, rolling updates and self-monitoring.

:warning: I am neither an Hashicorp employee nor a Kubernetes guru. You should first fork this repository, review the code, and remove things you don’t want or need. Don’t blindly use my configs unless you know what that entails.

## Prerequisites

- [Bash](https://www.gnu.org/software/bash/)
- [Helm](https://helm.sh/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)

## Usage

Feel free to edit the [values.yaml files](https://github.com/TommyStarK/vault-k8s/tree/main/vault/values.yaml) to fit to your needs.

```bash
❯ ./vk -h
vk - Vault toolkit for Kubernetes

Usage: vk [options] --deploy-agent
	vk [options] --render-template
	vk [options] --setup-cluster
	vk [options] --unseal
	vk [options] --delete

Options:
	-c | --cluster    Specify Kubernetes cluster
	-t | --target     Tools (agent, vault server)

Examples:
	# Render template for the Agent
	./vk --render-template --target=agent
	# Deploy Agent to minikube
	./vk --deploy-agent --cluster=minikube
	# Deploy the vault server cluster to Kubernetes cluster 'demo'
	./vk --setup-cluster --cluster=demo
	# Unseal Vault nodes
	./vk --unseal --cluster=demo
	# Delete Agent and Vault from Kubernetes cluster 'demo'
	./vk -c=demo -t=agent,vault --delete
```

## Demo using GKE + Minikube

To demonstrate how to use `vault-k8s`, we will use [GKE](https://cloud.google.com/kubernetes-engine) and [minikube](https://minikube.sigs.k8s.io/docs/). Feel free to use the cloud provider or whatever setup you want. Be aware of the required changes if you do so.

For demo purposes, the Vault cluster will be deployed **without** enabling data persistence, pod resources, ingress. See the [production readiness](https://github.com/TommyStarK/vault-k8s#production-readiness) section for more details regarding these topics.

### Cluster setup

First step, let's create the dedicated Kubernetes cluster for Vault.

```bash
❯ gcloud container clusters create vault-cluster --machine-type e2-standard-8
```

Once the cluster is ready, create the `vault` namespace

```bash
❯ kubectl create namespace vault
```

We can now proceed and deploy the Vault cluster

```bash
❯ ./vk --setup-cluster --cluster=<CLUSTER_NAME>
```

Retrieve the unseal keys and root token

```bash
❯ kubectl exec -ti -n vault vault-0  -- vault operator init
[...]
Unseal Key 1: X/LOC5Rp3xqj5hXx0WNKP3NEP7iTjev7nZu4odFowEnc
Unseal Key 2: +9w3RUIRQacDaA6OtQWpXinyzyxgI+ZnedyfM4WsK1VF
Unseal Key 3: nLQzt/CbMiGMUkpuD6lKtqYfB+wL7a6H41jqNM8TtI0r
Unseal Key 4: Tuu7gGO5b9bd+6+QgYy1QGUO4ct6QMdY2nXXvv0iragP
Unseal Key 5: McorsyqFP2PknmP6u45t86OYuWybledZa9IbfkUGBpIB

Initial Root Token: s.RHFKyNsi3gmXuki9D6MZIAn3
[...]
```

Pick 3 out of 5 unseal keys and export them like below

```bash
❯ export VAULT_UNSEAL_KEY_1="X/LOC5Rp3xqj5hXx0WNKP3NEP7iTjev7nZu4odFowEnc"
❯ export VAULT_UNSEAL_KEY_2="+9w3RUIRQacDaA6OtQWpXinyzyxgI+ZnedyfM4WsK1VF"
❯ export VAULT_UNSEAL_KEY_3="nLQzt/CbMiGMUkpuD6lKtqYfB+wL7a6H41jqNM8TtI0r"
[...]
```

Let's unseal all nodes of our Vault cluster

```bash
❯ ./vk --unseal --cluster=<CLUSTER_NAME>
```

Finally let's check the cluster state

```bash
# Login first with root token
❯ kubectl exec -ti -n vault vault-0 -- vault login s.RHFKyNsi3gmXuki9D6MZIAn3
# Now we can list the cluster members
❯ kubectl exec -ti -n vault vault-0 -- vault operator raft list-peers
```

### Agent injector

Sensitive data are not managed by Kubernetes itself. We are relying on a high-availability Vault cluster to handle that part.
Kubernetes is not secret aware, instead it is configured to authenticate to Vault and is granted access to a certain set of secrets.

We are using the vault agent injector to dynamically fetch secrets from our Vault cluster and inject them into
pods before scheduling.

Everything has been designed for being fully automated throughout a continuous integration and continuous deployment process. However, initial setup of the cluster requires specific operations to be performed beforehand.

First thing, the Vault Cluster must be up, running and unsealed.

Assuming this is the first time your are setting up the Kubernetes cluster. We need first to deploy the vault agent injector in our applicative cluster:

> When speaking about "applicative" cluster we mean the cluster holding your application.

> Production hardenning requirements for Vault require to have a dedicated infrastucture for it. Therefore we are configuring the vault agent to connect to our external dedicated HA Vault cluster.

For demo purposes we will use `minikube` to deploy the agent

```bash
❯ minikube start
```

```bash
❯ ./vk --deploy-agent --cluster=minikube
```

The Vault agent injector is now running, we need a few more steps
before switching to the vault side.

- Retrieve the token name bound to the vault agent service account

```bash
❯ VAULT_HELM_SECRET_NAME=$(kubectl get secrets -n xceed --output=json | jq -r '.items[].metadata | select(.name|startswith("vault-agent-injector-token-")).name')
```

- Retrieve the value of this token

```bash
❯ TOKEN_REVIEW_JWT=$(kubectl get secret -n xceed $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)
```

- Retrieve the Kubernetes host

```bash
❯ KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
```

- Retrieve the Kubernetes root CA certificate

```bash
❯ KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
```

We are ready to switch context and connect to the Vault cluster.
We must now, configure the Vault cluster to authorize our "applicative" cluster and allow our vault agent to fetch specific secret(s).

- Let's login into our Vault cluster

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault login
```

- We enable the auth Kubernetes method for a specific `<ENV>`. This way we could configure several Kubernetes clusters to access our Vault.

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault auth enable -path=<ENV> kubernetes
```

- Create a dedicated policy for the vault agent allowing access only for a subset of secrets.

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault policy write vault-agent-injector - <<EOF
path "secret/data/<ENV>/*" {
  capabilities = ["read"]
}

path "secret/metadata/<ENV>/*" {
  capabilities = ["read", "list"]
}
EOF
```

- Configure the Kubernetes auth method with the information retrieved previously from our applicative cluster

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault write auth/<ENV>/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        issuer="https://kubernetes.default.svc.cluster.local"
```

- Create a Kubernetes auth role and bind it to the vault agent policy and service account

:warning: The Vault agent injector is relying on its service account for being able to authenticate to our Vault. Service accounts are namespaced.

For CI/CD workflow, usually tests are run in Kubernetes preview environments, and thus dynamically generated namespaces.
In that situation we have to allow a fixed service account name for
any namespace ([doc](https://www.vaultproject.io/api/auth/kubernetes#bound_service_account_namespaces)), this way CI pipelines can create Kubernetes namespaces on-the-fly and all ephemeral components running in those namespaces can access secrets.

If you wish to restrict access to a specific namespace you must
edit the following and command with `bound_service_account_namespaces=<NAMESPACE>`.

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault write auth/<ENV>/role/vault-agent-injector \
        bound_service_account_names=vault-agent-injector \
        bound_service_account_namespaces='*' \
        policies=vault-agent-injector \
        ttl=24h
```

## Production readiness

- [Vault production hardening](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)
- [Kubernetes security consideration](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-security-concerns)
- [Performanace tuning](https://developer.hashicorp.com/vault/tutorials/operations/performance-tuning)
- [Integrated storage vs external storage](https://developer.hashicorp.com/vault/docs/configuration/storage#integrated-storage-vs-external-storage)

### Ingress

By default, ingress is disabled. `Vault` is accessible through Kubernetes `LoadBalancer`. There are prerequisites for being able to expose `HTTPS` route for this service.

You must have an Ingress controller to satisfy an ingress. Please be sure you have [Ingress-NGINX Controller](https://github.com/kubernetes/ingress-nginx/) running in your cluster. Feel free to use the ingress controller of your choice, but do not forget to update the ingress
template accordingly.

Follow the steps below if you want Vault to be accessible over `HTTPS` from outside the cluster.

1. Add the certificate and key to the secret template (file is located under `vault/templates/secret/vault-tls.yaml`)
2. Set the `ingress.enable` attribute to `true` in the according [values.yaml file](https://github.com/TommyStarK/vault-k8s/tree/main/vault/values.yaml)
3. Set the `ingress.host` attribute with your domain in the according [values.yaml file](https://github.com/TommyStarK/vault-k8s/tree/main/vault/values.yaml)
4. Render chart templates
