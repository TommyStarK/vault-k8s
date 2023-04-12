# vault-k8s

`vault-k8s` is an attempt to ease the setup of a Vault cluster running on Kubernetes. The aim of this repository is to facilitate the deployment of a high availability 5 nodes Vault cluster using the Raft integrated storage with end-to-end TLS. A tiny script `vk` within this repo will help to achieve that.

The idea is to have an ad-hoc, reproducible, configurable, extendable way of deploying a Vault cluster in a dedicated infrastructure. It also helps to deploy the agent injector on the Kubernetes cluster(s) of your choice, giving you the freedom to fetch secrets from your Vault and inject them before scheduling pods.

This repository is very opinionated and mostly built thanks to [hashicorp/vault-helm](https://github.com/hashicorp/vault-helm) and [hashicorp/vault-k8s](https://github.com/hashicorp/vault-k8s).

You will find a [section](https://github.com/TommyStarK/vault-k8s#production-readiness) gathering the different links I found useful to decide how to properly deploy this stack to be used in production and meet my needs/constraints. It covers things like capacity planning, pod resources, data persistence, using Ingress, rolling updates and self-monitoring.

:warning: I am neither an Hashicorp employee nor a Kubernetes guru. You should first fork this repository, review the code, and remove things you don’t want or need. Don’t blindly use my configs unless you know what that entails.

## Prerequisites

- [Bash](https://www.gnu.org/software/bash/)
- [Helm](https://helm.sh/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)

## Usage

Feel free to edit the [Vault values.yaml file](https://github.com/TommyStarK/vault-k8s/tree/main/vault/values.yaml) to fit to your needs.

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

## Demo using GKE

To demonstrate how to use `vault-k8s`, we will use [GKE](https://cloud.google.com/kubernetes-engine). Feel free to use the cloud provider or whatever setup you want. Be aware of the required changes if you do so.

For demo purposes, the Vault cluster will be deployed **WITHOUT** enabling data persistence, pod resources, ingress. See the [production readiness](https://github.com/TommyStarK/vault-k8s#production-readiness) section for more details regarding these topics.

### TLS setup

Generate the Vault server private key:

```bash
❯ openssl genrsa -out vault/tls/vault.key 2048
```

Generate the certificate signing request:

```bash
❯ openssl req -new -key vault/tls/vault.key -out vault/tls/vault.csr -config vault/tls/vault.conf
```

Encode the certificate signing request to base64:

```bash
❯ echo -n "$(cat vault/tls/vault.csr)" | base64 | tr -d '\n'
```

Copy/paste the output [here](https://github.com/TommyStarK/vault-k8s/blob/main/vault/tls/csr.yaml#L10).

### Cluster setup

First step, let's create the dedicated Kubernetes cluster for Vault:

```bash
❯ gcloud container clusters create vault-cluster --machine-type e2-standard-8
```

Once the cluster is ready, create the `vault` namespace:

```bash
❯ kubectl create namespace vault
```

Send the certificate signing request to Kubernetes:

```bash
❯ kubectl apply -f vault/tls/csr.yaml
```

Approve the certificate signing request:

```bash
❯ kubectl certificate approve vault-internal.svc
```

Retrieve the certificate:

```bash
❯ kubectl get csr vault-internal.svc -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out vault/tls/vault.crt
```

Retrieve the CA certificate:

```bash
❯ kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d > vault/tls/vault.ca
```

Create a Kubernetes secret holding the TLS artifacts:

```bash
❯ kubectl create secret generic vault-tls -n vault --from-file=vault.key=vault/tls/vault.key --from-file=vault.crt=vault/tls/vault.crt --from-file=vault.ca=vault/tls/vault.ca
```

We can now proceed and deploy the Vault cluster:

```bash
❯ ./vk --setup-cluster --cluster=<VAULT_CLUSTER_NAME>
```

Retrieve the unseal keys and root token:

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

Pick 3 out of 5 unseal keys and export them like below:

```bash
❯ export VAULT_UNSEAL_KEY_1="X/LOC5Rp3xqj5hXx0WNKP3NEP7iTjev7nZu4odFowEnc"
❯ export VAULT_UNSEAL_KEY_2="+9w3RUIRQacDaA6OtQWpXinyzyxgI+ZnedyfM4WsK1VF"
❯ export VAULT_UNSEAL_KEY_3="nLQzt/CbMiGMUkpuD6lKtqYfB+wL7a6H41jqNM8TtI0r"
```

Let's unseal all nodes of our Vault cluster:

```bash
❯ ./vk --unseal --cluster=<VAULT_CLUSTER_NAME>
```

:warning: Some nodes may restart in between, if it happens, run the unseal command again.

Finally let's check the cluster state:

```bash
# Login first with root token
❯ kubectl exec -ti -n vault vault-0 -- vault login

# Now we can list the cluster members
❯ kubectl exec -ti -n vault vault-0 -- vault operator raft list-peers
Node       Address                        State       Voter
----       -------                        -----       -----
vault-0    vault-0.vault-internal:8201    leader      true
vault-1    vault-1.vault-internal:8201    follower    true
vault-2    vault-2.vault-internal:8201    follower    true
vault-3    vault-3.vault-internal:8201    follower    true
vault-4    vault-4.vault-internal:8201    follower    true
```

You can access the Vault UI by running the following command:

```bash
❯ open "http://$(kubectl get -n vault service vault-active| awk 'NR>1 {print $4}'):8200/ui"
```

At that point the cluster is up, running and unsealed, the only thing we need is to retrieve the endpoint to communicate with our Cluster

```bash
❯ kubectl get -n vault service vault-active| awk 'NR>1 {print $4}'
34.91.249.155
```

### Agent injector

Sensitive data are not managed by Kubernetes itself. We are relying on a HA Vault cluster to handle that part.
Kubernetes (thanks to the Vault agent injector) is configured to authenticate to Vault and is granted access to a certain set of secrets. It allows to dynamically fetch secrets from our Vault and inject them into pods before scheduling.

Everything has been designed for being fully automated throughout a continuous integration and continuous deployment process. However, initial setup of the cluster requires specific operations to be performed beforehand.

First thing, the Vault cluster must be up, running and unsealed.

Assuming this is the first time your are setting up the Kubernetes applicative cluster. We need first to deploy the vault agent injector.

> When speaking about "applicative" cluster we mean the cluster holding your application.
> Production hardenning requirements for Vault require to have a dedicated infrastucture for it. Therefore we are configuring the vault agent to connect to our external dedicated HA Vault cluster.

Let's create the Kubernetes app cluster:

```bash
❯ gcloud container clusters create app-cluster --machine-type e2-small --disk-size 10
```

Once the cluster is ready, create the `vault` namespace:

```bash
❯ kubectl create namespace vault
```

Before deploying the agent, we must update its [values.yaml](https://github.com/TommyStarK/vault-k8s/blob/main/agent/values.yaml#L10) with the endpoint of the Vault cluster. This way the agent will be able to communicate with it. For demo purposes, the Vault cluster has been deployed **WITHOUT** enabling ingress so we use the `service LoadBalancer IP`.

We retrieved it just before, we can update the `vault.endpoint` value with `https://34.91.249.155:8200`.

We can now proceed and deploy the Vault agent injector:

```bash
❯ ./vk --deploy-agent --cluster=<APP_CLUSTER_NAME>
```

The Vault agent injector is now running, we need a few more steps before switching to the vault side.

- Retrieve the token name bound to the vault agent service account:

```bash
❯ VAULT_HELM_SECRET_NAME=$(kubectl get secrets -n vault --output=json | jq -r '.items[].metadata | select(.name|startswith("vault-token-")).name')
```

- Retrieve the value of this token:

```bash
❯ TOKEN_REVIEW_JWT=$(kubectl get secret -n vault $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)
```

- Retrieve the Kubernetes host:

```bash
❯ KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
```

- Retrieve the Kubernetes root CA certificate:

```bash
❯ KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
```

We are ready to switch context and connect to the Vault cluster.
We must now, configure the Vault cluster to authorize our "applicative" cluster and allow our vault agent to fetch specific secret(s).

Let's login into our Vault cluster:

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault login
```

We enable the auth Kubernetes method for a specific `<ENV>`. This way we could configure several Kubernetes clusters to access our Vault.
For demo purposes we will use the path `demo`, Feel free to set what you want.

:warning: We are going to replace `<ENV>` by `demo` in all following commands. If you wish to set another path DO NOT forget to update the `environment` value in the agent [values.yaml](https://github.com/TommyStarK/vault-k8s/blob/main/agent/values.yaml#L2). This value is used in the `AGENT_INJECT_VAULT_AUTH_PATH`.

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault auth enable -path=<ENV> kubernetes
```

Create a dedicated policy for the vault agent allowing access only for a subset of secrets:

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault policy write vault-agent-injector - <<EOF
path "<ENV>/metadata/*" {
  capabilities = ["read", "list"]
}

path "<ENV>/data/*" {
  capabilities = ["read", "list"]
}
EOF
```

Configure the Kubernetes auth method with the information retrieved previously from our applicative cluster:

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault write auth/<ENV>/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        issuer="https://kubernetes.default.svc.cluster.local"
```

Create a Kubernetes auth role and bind it to the vault agent policy and service account:

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault write auth/<ENV>/role/vault-agent-injector \
        bound_service_account_names=vault-agent-injector \
        bound_service_account_namespaces='*' \
        policies=vault-agent-injector \
        ttl=24h
```

:warning: The Vault agent injector is relying on its service account for being able to authenticate to our Vault. Service accounts are namespaced.

For CI/CD workflow, usually tests are run in Kubernetes preview environments, and thus dynamically generated namespaces.
In that situation we have to allow a fixed service account name for
any namespace ([doc](https://www.vaultproject.io/api/auth/kubernetes#bound_service_account_namespaces)), this way CI pipelines can create Kubernetes namespaces on-the-fly and all ephemeral components running in those namespaces can access secrets.

If you wish to restrict access to a specific namespace you must
edit the following and command with `bound_service_account_namespaces=<NAMESPACE>`.

### Fetch secret

The setup of both the Vault cluster and the agent injector are done. Your applicative cluster should be able to fetch secrets from the Vault thanks to the agent injector.

To demonstrate that, we are going to create a demo secret and deploy a demo app to test everything is working as expected.

First, let's enable the KV v2 secret engine:

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault secrets enable -path=demo kv-v2
```

Then, create a secret:

```bash
❯ kubectl exec -ti vault-0 -n vault -- vault kv put demo/dummy DEMO_SECRET="this is secret"
```

Now switch context to interact with the "applicative" cluster.
Let's deploy a demo app to test fetching secret from the Vault:

```bash
❯ kubectl apply -f demo/demo.yaml
```

Run this last command to ensure it is working :smile:

```bash
❯ kubectl logs -n vault $(kubectl get pod -n vault -l app=demo -o jsonpath="{.items[0].metadata.name}") -c demo
```

### Cleanup

- remove vault stack

```bash
❯ ./vk --delete --target=agent --cluster=<APP_CLUSTER_NAME>
❯ ./vk --delete  --target=vault --cluster=<VAULT_CLUSTER_NAME>
```

- remove clusters

```bash
❯ gcloud container clusters delete app-cluster
❯ gcloud container clusters delete vault-cluster
```

## Production readiness

- [Vault security model](https://developer.hashicorp.com/vault/docs/internals/security)
- [Vault production hardening](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)
- [Kubernetes security consideration](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-security-concerns)
- [Performance tuning](https://developer.hashicorp.com/vault/tutorials/operations/performance-tuning)
- [Integrated storage vs external storage](https://developer.hashicorp.com/vault/docs/configuration/storage#integrated-storage-vs-external-storage)

### Ingress

By default, ingress is disabled. `Vault` is accessible through Kubernetes `LoadBalancer`. There are prerequisites for being able to expose `HTTPS` route for this service.

You must have an Ingress controller to satisfy an ingress. Please be sure you have [Ingress-NGINX Controller](https://github.com/kubernetes/ingress-nginx/) running in your cluster. Feel free to use the ingress controller of your choice, but do not forget to update the ingress template accordingly.

Follow the steps below if you want Vault to be accessible over `HTTPS` from outside the cluster.

Let's assume your are using the following domain: `https://vault.cluster.com`

- On Vault side:

1. Add the certificate and key to the secret [template](https://github.com/TommyStarK/vault-k8s/blob/main/vault/templates/secret/vault-gateway-tls.yaml).
2. Set the `ingress.enable` attribute to `true` in the according [values.yaml file](https://github.com/TommyStarK/vault-k8s/blob/main/vault/values.yaml#L4).
3. Set the `ingress.host` attribute with your domain (vault.cluster.com) in the according [values.yaml file](https://github.com/TommyStarK/vault-k8s/blob/main/vault/values.yaml#L5).

- On the agent injector side:

1. Create a secret holding the CA certificate used to issue the certificate for `vault.cluster.com`.
2. Update the [`vault.endpoint`](https://github.com/TommyStarK/vault-k8s/blob/main/agent/values.yaml#L10) with `https://vault.cluster.com`
3. Comment this [line](https://github.com/TommyStarK/vault-k8s/blob/main/demo/demo.yaml#L19).
4. Uncomment these [lines](https://github.com/TommyStarK/vault-k8s/blob/main/demo/demo.yaml#L20-L21).

:warning: When creating the secret containing the CA certificate used to issue the certificate for `https://vault.cluster.com` be sure it matches the name used in the [annotation](https://github.com/TommyStarK/vault-k8s/blob/main/demo/demo.yaml#L20) and the secret name specified in the [annotation](https://github.com/TommyStarK/vault-k8s/blob/main/demo/demo.yaml#L21).

For example, based on the `demo.yaml`:

```bash
❯ kubectl create secret generic vault-ca -n vault --from-file=vault.ca=<PATH_TO_CA_CERT>
```
