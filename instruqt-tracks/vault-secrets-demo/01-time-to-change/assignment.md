---
slug: time-to-change
id: m7dqzx25bhi1
type: challenge
title: Time to Change
teaser: Busted! Secrets at HashiCups, Inc. are not centralized or governed. It's time
  to change.
notes:
- type: text
  contents: |-
    You are a DevOps engineer at HashiCups, Inc.

    HashiCups has recently been audited by regulators who determined that the
    HashiCups Store app has a poor secrets management posture. For example,
    the auditors found secrets for the HashiCups app in plain text in git
    repos, shared mounts, and spreadsheets.

    To satisfy the regulators and auditors, HashiCups has decided to adopt
    Vault as a centralized secrets management service. Your team has been
    tasked with migrating from your legacy secrets storage to the new Vault
    secrets management solution.
- type: text
  contents: |-
    Vault is a tool for securely accessing secrets. A secret is anything that
    you want to tightly control access to, such as an API key, password, or
    certificate.

    Vault provides a unified interface to any secret, while providing tight
    access control and recording a detailed audit log.
- type: text
  contents: |-
    Two important Vault components are used in this track:

    - **Auth Methods**
      - Perform authentication and are responsible for assigning identity
      (and policies) to a user.
    - **Secrets Engines**
      - Are components which store, generate, or encrypt secrets.

    You can visually see how these interact on the next note.
- type: text
  contents: '![Vault Triangle](https://github.com/hashicorp/field-demo-vault-secrets-mgmt/raw/main/images/vault-triangle.png)'
- type: text
  contents: |-
    The HashiCups Store itself consists of several different components, all
    running within Kubernetes. Many of these components require a secret to
    interact with another component.
- type: text
  contents: |-
    The first components we'll look at are:

      - [`frontend`](https://github.com/hashicorp-demoapp/frontend)
        - A front end web service for displaying HashiCups products.
      - [`product-api`](https://github.com/hashicorp-demoapp/product-api-go)
        - A REST API service abstracting the `products` database in `postgres`
        - _Requires credentials to connect to `postgres`_
      - [`postgres`](https://github.com/hashicorp-demoapp/postgres)
        - A PostgreSQL database that contains the HashiCups products

    On the next note, you can see how all the components interact. Note that Consul is used for service discovery.
    Don't worry about remembering it all now, we'll address them
    each in this track.
- type: text
  contents: '![HashiCups Reference Architecture Diagram](https://github.com/hashicorp/field-demo-vault-secrets-mgmt/raw/main/images/infra-new.png)'
- type: text
  contents: |-
    Your team has already deployed all of the components of the HashiCups
    Store into Kubernetes, and one of your team members has already configured
    the [`kubernetes` auth method](https://www.vaultproject.io/docs/auth/kubernetes/)
    in Vault.

    The `kubernetes` auth method can be used to authenticate with Vault using a
    Kubernetes Service Account Token. This method of authentication makes it easy
    to introduce a Vault token into a Kubernetes Pod.

    Because this has already been configured, you don't have to worry about how
    the Kubernetes pods in the next steps are authenticating to Vault.
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes
- title: Vault UI
  type: service
  hostname: kubernetes
  path: /ui
  port: 30082
- title: HashiCups Store
  type: service
  hostname: kubernetes
  path: /
  port: 30090
difficulty: basic
timelimit: 1320
---

The [root token](https://www.vaultproject.io/docs/concepts/tokens#root-tokens) for
this Vault installation is simply "root". Using the root token, you'll have
full access to Vault to begin this challenge. You can log in using the
[`vault login`](https://www.vaultproject.io/docs/commands/login/) command,
or using the token in the UI.

---

Though your team has already deployed all of the components of the HashiCups
Store into Kubernetes, when you open up the HashiCups Store tab,
you see the app is showing "`Error :(`".

The secrets used to connect each component must not have been migrated
over to Vault correctly. You know your team member has already migrated the
secret into Vault, but you don't know at what path it is stored.

Everything in Vault is path based, so the first thing you will need to do
is find out at what path the `kv` secrets engine is mounted.

```bash
vault secrets list
```

Notice that the `kv` secrets engine is mounted at the path `kv/`. Next, you
need to list the keys of that `kv` mount until you find the Customer Profile
Postgres database credentials.

```bash
vault kv list kv
```

Note that there is a key inside the `kv` mount already (`db`). Continue digging
using `vault kv list` to get familiar with path based secrets.

```bash
vault kv list kv/db
vault kv list kv/db/postgres
```

You can also open up the Vault UI tab, type "root" into the Token auth method, and
log in to visually browse the `kv` mount.

Since you know the `products-api` needs to talk to the `postgres` service,
you should start there. You can read a secret with the `read` command.

```bash
vault read kv/db/postgres/product-db-creds
```

Something about the password doesn't look right...

Now that you have an idea of how path based secrets are laid out in Vault's
`kv` secrets engine, let's start fixing the HashiCups store.
