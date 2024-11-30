## Hashicorp Vault integration to k8s vault is alternative to AWS secret manager

Create the vault namespace:

```bash
kubectl create namespace vault
```

Add the HashiCorp Helm repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Install HashiCorp Vault using Helm:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

# Unseal vault (Non development mode)
```
kubectl exec -ti vault-0 -n vault -- /bin/sh
vault operator init
copy all 5 Unseal key and Root Token

Do this 3x
vault operator unseal: [supply unseal key from 1-3]

and finally do login and use Initial Root Token as credential

vault login

```
Create a policy for reading secrets (read-policy.hcl):

```bash
cat <<EOF > /home/vault/read-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF
```

Write the policy to Vault:

```bash
vault policy write read-policy /home/vault/read-policy.hcl
```

Enable Kubernetes authentication in Vault:

```bash
vault auth enable kubernetes
```

Configure Vault to communicate with the Kubernetes API server
Trust me don't change the following lines below, I encounter lot of issues on this!

```bash
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```
Just in case you are wondering where this environment variable came from KUBERNETES_PORT_443_TCP_ADDR 
this was injected by k8s to a pod

```
printenv 
```

Create a role(read-only-from-vault) that binds the above policy to a Kubernetes service account(vault-svc-account) in a specific namespace. This allows the service account to access secrets stored in Vault

bound_service_account_namespaces=whatever_ns_your_kick_ass_apps_resides

```bash
vault write auth/kubernetes/role/read-only-from-vault \
   bound_service_account_names=vault-svc-account,dev-vault-svc-account,prod-vault-svc-account \
   bound_service_account_namespaces=development,production \
   policies=read-policy \
   ttl=1h
```

Create secret and activate secret/kv module

```bash
vault secrets enable -path=secret kv
vault kv put secret/dbcred host=192.168.99.4 db=development username=demo password=password
```

Verify if secret created or not

```bash
vault kv list secret
```
# Create SA account it should reside whatever your app namespace it's a must!

```
kubectl apply -f sa.yaml
```

# Deploy your pod 
```
kubectl apply -f my-app.yaml 

learn-vault git:(master) kubectl get pods -n development
NAME                          READY   STATUS    RESTARTS   AGE
my-app                        2/2     Running   0          55m

Don't get confuse why there is 2/2 pod the vault was sidecar to your my-app pod when I have time
i'm going to implement this using initContainer approach without sidecar in the app
```
