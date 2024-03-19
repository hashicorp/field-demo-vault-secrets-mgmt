---
slug: vault-agent-with-kubernetes
id: nppvv9w2qx4v
type: challenge
title: Vault Agent with Kubernetes
teaser: Modify a Kubernetes deployment file to leverage the Vault agent and inject
  secrets.
notes:
- type: text
  contents: |-
    HashiCorp provides a tool called
    [`vault-k8s`](https://github.com/hashicorp/vault-k8s) which leverages the
    Kubernetes Mutating Admission Webhook to intercept and augment
    specifically annotated pod configuration for secret injection using Init
    and Sidecar containers.

    Applications need only concern themselves with finding a secret at a
    filesystem path, rather than managing tokens, connecting to an external
    API, or other mechanisms for direct interaction with Vault.
- type: text
  contents: |-
    For this challenge, you don't have to worry about installing the sidecar
    injector. Instead, you will have to modify an existing Kubernetes
    deployment spec file to read from the path that you just created in the
    previous challenge.
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes
- title: k8s Dir
  type: code
  hostname: kubernetes
  path: /root/k8s/
- title: Policies Dir
  type: code
  hostname: kubernetes
  path: /root/policies/
- title: Vault UI
  type: service
  hostname: kubernetes
  path: /ui/
  port: 30082
- title: K8s UI
  type: service
  hostname: kubernetes
  port: 30443
- title: HashiCups Store
  type: service
  hostname: kubernetes
  path: /
  port: 30090
difficulty: basic
timelimit: 1320
---

The Products API deployment that you have been manipulating over the course
of this track is located in `products-api.yml` in the "k8s Dir" tab. Open
it up and take a look.

Look around for the string `agent-inject`. You'll notice a reference to a path
to read from Vault, and a template file that will be written to the host
Vault is deployed on.

You need to update the path to reflect the one you created in the last
challenge, `database/creds/products-api`.

You can do that manually, or you can use this quick one-liner below, once
you understand what you're doing with this command.

```run
sed -i 's/kv\/db\/postgres\/product-db-creds/database\/creds\/products-api/g' /root/k8s/products-api.yml
```

Update the deployment in Kubernetes.

```bash,run
kubectl apply -f k8s/products-api.yml
```

This will not be enough to get HashiCups back online, though. The
`products-api` policy needs to be updated to reflect the new path the
`products-api-deployment` will be reading from.

Remembering what you learned about ACLs earlier in this track, modify
`/root/policies/products-api-policy.hcl` and update the `products-api`
policy in Vault to reflect the new path it should have access to read from.
Open up `/root/policies/products-api-policy.hcl` and make it reflect the below
policy.

```nocopy
path "database/creds/products-api" {
  capabilities = ["read"]
}
```

Once again, there is a one-liner you can use to achieve this.

```run
sed -i 's/kv\/db\/postgres\/product-db-creds/database\/creds\/products-api/g' /root/policies/products-api-policy.hcl
```

You'll then need to upload your changes to Vault.

```bash,run
vault policy write products-api policies/products-api-policy.hcl
```

Restart the deployment one last time.

```bash,run
kubectl rollout restart deployment products-api-deployment
```

Run `kubectl get deploy` until the products-api-deployment is ready.

You should now be able to refresh the HashiCups UI and see the products.
You are now using dynamic credentials, and the root credentials are no
longer known by anyone on your team.
