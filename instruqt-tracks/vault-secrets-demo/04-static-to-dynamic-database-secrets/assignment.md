---
slug: static-to-dynamic-database-secrets
id: vzosygehzqnv
type: challenge
title: Static to Dynamic Database Secrets
teaser: No more shared passwords - set up Vault to generate dynamic secrets for the
  Products database.
notes:
- type: text
  contents: |-
    Now, you already moved the database credentials for the Products API from
    a file to Vault, but it would be even better if you didn't have to use
    the same credentials, and could generate them on the fly.
- type: text
  contents: |-
    Unlike the KV secrets engine where you had to put data into the store
    yourself, dynamic secrets are generated when they are accessed. Dynamic
    secrets do not exist until they are read, so there is no risk of someone
    stealing them or another client using the same secrets.
- type: text
  contents: |-
    With every dynamic secret, Vault creates a _lease_: metadata containing
    information such as TTL, renewability, and more. When a lease expires,
    Vault automatically revokes the dynamic secret, and the consumer of the
    secret can no longer be certain that it is valid. This minimizes the
    amount of time the secret exists.
- type: text
  contents: |-
    The `database` secrets engine can generate database credentials
    dynamically based on configured roles. With each request, a unique
    username and password pair is generated and returned.

    That will be the focus of this challenge.
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes
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
In order to generate dynamic secrets for Postgres, the first thing you'll
need to enable is the `database` secrets engine.

```
vault secrets enable database
```

Once it's enabled, we need to configure it to communicate with the Postgres
database. We'll use the credentials that we put in the `kv` engine earlier
in the next step.

```
vault read kv/db/postgres/product-db-creds
```

With the credentials fresh in mind, you must configure the `database`
secrets engine to communicate with your Postgres server. You'll need to
get the Cluster IP for the Postgres database, and you should also set the
PG_USER and POSTGRESS_PASS variables for easy of use and clarity..

```
export PG_HOST=$(kubectl get svc -l app=postgres -o=json | jq -r '.items[0].spec.clusterIP')
export PG_USER=postgres
export PG_PASS=password
```

```
vault write database/config/hashicups-pgsql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="products-api" \
    connection_url="postgresql://{{username}}:{{password}}@$PG_HOST:5432/?sslmode=disable" \
    username=$PG_USER \
    password=$PG_PASS
```

Next, you'll have to configure a role in the `database` secrets engine and associate
it with a `CREATE ROLE` statement in Postgres. Here you can also configure things like
the `default_ttl` or `max_ttl`, which refers to the duration of the lease on the
secrets before they expire and are automatically revoked.

```
vault write database/roles/products-api \
    db_name=hashicups-pgsql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' SUPERUSER; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="30m" \
    max_ttl="1h"
```

You are now ready to generate dynamic Postgres credentials.

```
vault read database/creds/products-api
```

Confirm that you can connect to Postgres with these credentials by plugging
the response values into the below command. Enter the generated password
when prompted.

```
psql -U <YOUR_GENERATED_USERNAME> -h $PG_HOST products
```

If you can successfully log in,  then you have successfully configured Vault
for dynamic database credentials and are ready to move on.

Exit `psql` by typing `\q`.

The last thing you want to do is rotate the root credential that you
configured the database secret engine with, since you don't want anyone
to be able to use that set of credentials again.

```
vault write -force database/rotate-root/hashicups-pgsql
```

Confirm that you can no longer login with the credentials as before, typing
"password" when prompted for the password of the postgres user.

```
psql -U postgres -h $PG_HOST
```

Now, you can delete the old path, since we no longer need it.

```
vault kv delete kv/db/postgres/product-db-creds
```

Sicne you have removed the KV path that was being used by the old Products
API deployment, you should remove the old Products API deployment
altogether. This is because the old deployment has cached results using
the old username and password. Don't worry, you'll create a new one in the
next challenge.

```
kubectl delete deployment products-api-deployment
```

Confirm the Products API deployment has been deleted.

```
kubectl get deployment
```

You should no longer see `products-api`, and if you refresh the HashiCups
Store tab, it should be showing `Error :(` again. You'll fix this in the
next challenge.

You have successfully configured Vault for dynamic secrets, and eliminated another
potential security issue by deleting the old credentials and rotating them in Vault.