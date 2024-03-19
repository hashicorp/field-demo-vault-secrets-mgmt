---
slug: restricting-access-to-secrets-using-acls
id: kdetbxsdjtsi
type: challenge
title: Restricting Access to Secrets Using ACLs
teaser: Now that you've fixed your first Vault secret, make sure only the right identities
  can read it.
notes:
- type: text
  contents: |-
    Vault centrally secures, stores, and controls access to secrets across
    distributed infrastructure and applications, so it is critical to control
    permissions before any user or machine can gain access.
- type: text
  contents: |-
    Vault uses policies to govern the behavior of clients and implements
    Role-Based Access Control (RBAC) by specifying access privileges (_authorization_).
    Vault supports both [ACL policies](https://www.vaultproject.io/docs/concepts/policies.html)
    and [Sentinel policies](https://www.vaultproject.io/docs/enterprise/sentinel/index.html).

    This challenge focuses on ACL policies.
- type: text
  contents: |-
    When you first initialize Vault, the root policy is created by default. The
    [root policy](https://www.vaultproject.io/docs/concepts/policies/#root-policy)
    is a special policy that gives superuser access to everything in Vault.
    This allows the superuser to set up the initial policies, auth methods,
    etc.

    This is the policy you have been using up until now.
- type: text
  contents: |-
    Since Vault enforces the principle of "least-privileged access" you should
    scope the access to be as limited as possible.

    As you have already seen - everything in Vault is path based.

    Admins write policies to grant or forbid access to certain paths and operations
    in Vault. Vault operates on a *secure by default standard*, and as such, an empty
    policy grants no permissions in the system.
- type: text
  contents: |-
    There are two roles at HashiCups that need access to these secrets.
    The DBA team (`dba-operator`) needs to be able to perform all CRUD operations
    on the Postgres creds path, and the Products API (`products-api`) only needs
    to read the secrets at that path.
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes
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

There are two primary roles that need to access the Customer Profile database: DBAs
(`dba-operator`) and the Products API (`products-api`) itself. The DBAs should be able to
perform all operations on the `kv/database/` path. The web service should only be able
to read secrets at that path.

You have been provided with pre-written ACL policies - one for the DBA operator
and one for the HashiCups Products API. You can find them in the "Policies Dir"
tab.

The products-api-policy.hcl file was already used by a setup script in the first challenge to create the `products-api` policy in Vault since it was needed to fix the HashiCups app in the previous challenge.

Once you understand the policies, write the `dba-operator` policy to Vault.

```bash,run
vault policy write dba-operator /root/policies/dba-operator-policy.hcl
```

---

Now that both of the required policies are in place for your users and applications, you
should enable the [Userpass auth method](https://www.vaultproject.io/docs/auth/userpass.html)
and create some users for that auth method that leverage the policies
you just created.

```bash,run
vault auth enable userpass
```

Create a user for Dan on the DBA team - call him `dba-dan`, and make sure that he gets
the `dba-operator` policy.

```bash,run
vault write auth/userpass/users/dba-dan password=dba-dan policies=dba-operator
```

Now that the user is created and configured with the right policy, you should log in
with the `dba-dan` user and confirm the policies are correctly applied. Before you do
that, you'll have to unset the `VAULT_TOKEN` environment variable, otherwise it will
take precedence over the `login` operation you are about to perform.

```bash,run
unset VAULT_TOKEN
vault login -method=userpass username=dba-dan password=dba-dan
```

Note the policy after logging in. Since you have logged in with the DBA user,
you should be able to read the secret from the `kv/db/*` path, but not from any
other path.

```bash,run
vault read kv/db/postgres/product-db-creds
vault read kv/not-db/
```

If you can read the credentials but get denied on the `not-db` path, you have
successfully set up the DBA Operator ACLs.
