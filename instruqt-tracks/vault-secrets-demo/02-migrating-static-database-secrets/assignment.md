---
slug: migrating-static-database-secrets
id: fnn7aesf1eov
type: challenge
title: Migrating Static Database Secrets
teaser: Migrate your first secret from a shared mount to Vault.
notes:
- type: text
  contents: |-
    The first secret that needs your attention is the credential set for the
    Products database. These credentials are used by the HashiCups Store's
    Product API service to pull in all of the products available to consumers.

    The credentials are currently stored in a text file on a shared mount.
    All of your team members at HashiCups have access to this shared mount,
    and whenever you need to deploy the app, a DBA SSHes into a box, references
    the file in the shared mount, and updates the app.
- type: text
  contents: |-
    This has caused a major issue for the auditors. You need to make sure that
    the credentials have been migrated correctly from the filesystem to Vault.
    Once you've done that, you need to make sure that only the identities that
    need access can access the secret, and then you should make the secret
    dynamic.

    More on that later.

    Your first task is to make sure those shared database credentials are correctly
    input into Vault.
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes
- title: Shared Drive
  type: code
  hostname: kubernetes
  path: /share/
- title: k8s Dir
  type: code
  hostname: kubernetes
  path: /root/k8s/
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
When the HashiCups Store was audited, the auditors found, among other
things, that the Postgres Products database credentials were clearly
visible to all internal team members on a shared drive, at
`/share/postgres-product-creds.txt`.

This file is viewable in the "Shared Drive" tab.
Noting that what you read in the shared file and what is in Vault
do not match, you should make an update to the path in Vault with
the right credentials.

```bash
vault kv put kv/db/postgres/product-db-creds \
  username="postgres" \
  password="password"
```

Once you have made the updates, you should redeploy the app so that it
pulls the latest credentials.

As noted earlier, you are running everything in Kubernetes, so you
will leverage the
[Vault Agent Sidecar Injector](https://www.vaultproject.io/docs/platform/k8s/injector/)
for Kubernetes. The Vault Agent leverages the
[Kubernetes auth method](https://www.vaultproject.io/docs/auth/kubernetes/)
to authenticate, and then injects secrets in a few ways.

You don't have to worry about how this is set up for now,
but you should understand what the containers it can inject do.

Two types of Vault Agent containers can be injected: _init_ and _sidecar_.

The init container will prepopulate the shared memory volume with the
requested secrets prior to the other containers starting. The sidecar
container will continue to authenticate and render secrets to the same
location as the pod runs. Using annotations, the initialization and
sidecar containers may be disabled.

At a minimum, every container in the pod will be configured to mount a
shared memory volume. This volume is mounted to `/vault/secrets` and will
be used by the Vault Agent containers for sharing secrets with the other
containers in the pod.

You can see where your team specifies the path for the secret if you look
inside `products-api.yml` file, which you can find under the "k8s
Dir" tab.

Look for `spec -> template -> metadata -> annotations -> vault.hashicorp.com.*`
to understand what is being read from Vault, and written to the local filesystem.

Look for spec -> template -> metadata -> annotations -> vault.hashicorp.com.*
to understand what is being read from Vault, and written to the local filesystem.

Now that you understand how the Vault Agent Sidecar Injector for Kubernetes
works, it's time to cycle the deployment and read in the new secrets using
the init container.
Since we are running all of our components in Kubernetes, we can do that with
[`kubectl`](https://kubernetes.io/docs/reference/kubectl/overview/).

You can see all of the deployments in your Kubernetes cluster.

```bash
kubectl get deploy
```

You should restart the `products-api-deployment`, you can do that with a single
command.

```bash
kubectl rollout restart deployment products-api-deployment
```

Keep checking the deployment status until it's up and running.

```bash
kubectl get deploy
```

Once the deployment has been restarted, open up the HashiCups Store tab again
and refresh the page. You should be able to see one of the HashiCups products now.

Now that you've migrated the credentials from the old shared file into Vault,
remove it.

```bash
rm /share/postgres-product-creds.txt
```

Now that the secret is protected by Vault, you've taken one step in the right
direction towards good secrets management hygiene.
