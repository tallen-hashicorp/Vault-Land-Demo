# Vault-Land-Demo
- [Instruqt Track 1](https://play.instruqt.com/hashicorp/tracks/vault-managing-secrets-and-moving-to-cloud)
- [Demo Script 1](https://docs.google.com/document/d/19WCDz3bvXDtkF24OLGBlKSdOrw_W6NLaBzRJC4kRHKw/edit)

You are a DevOps engineer at HashiCups, Inc.

HashiCups has recently been audited by regulators who determined that the HashiCups Store app has a poor secrets management posture. For example, the auditors found secrets for the HashiCups app in plain text in git repos, shared mounts, and spreadsheets.

To satisfy the regulators and auditors, HashiCups has decided to adopt Vault as a centralized secrets management service. Your team has been tasked with migrating from your legacy secrets storage to the new Vault secrets management solution.

Vault is a tool for securely accessing secrets. A secret is anything that you want to tightly control access to, such as an API key, password, or certificate.

Vault provides a unified interface to any secret, while providing tight access control and recording a detailed audit log.

Two important Vault components are used in this track:

* Auth Methods
    * Perform authentication and are responsible for assigning identity (and policies) to a user.
* Secrets Engines
    * Are components which store, generate, or encrypt secrets.

The HashiCups Store itself consists of several different components, all running within Kubernetes. Many of these components require a secret to interact with another component.

The first components we'll look at are:

* `frontend`
    * A front end web service for displaying HashiCups products.
* `product-api`
    * A REST API service abstracting the products database in postgres
    * Requires credentials to connect to postgres
* `postgres`
    * A PostgreSQL database that contains the HashiCups products

On the next note, you can see how all the components interact. Note that Consul is used for service discovery. Don't worry about remembering it all now, we'll address them each in this track.

Your team has already deployed all of the components of the HashiCups Store into Kubernetes, and one of your team members has already configured the kubernetes auth method in Vault.

The kubernetes auth method can be used to authenticate with Vault using a Kubernetes Service Account Token. This method of authentication makes it easy to introduce a Vault token into a Kubernetes Pod.

Because this has already been configured, you don't have to worry about how the Kubernetes pods in the next steps are authenticating to Vault.

---

# Time to Change
The root token for this Vault installation is simply "root". Using the root token, you'll have full access to Vault to begin this challenge. You can log in using the `vault login` command, or using the token in the UI.

Though your team has already deployed all of the components of the HashiCups Store into Kubernetes, when you open up the HashiCups Store tab, you see the app is showing "`Error :(`".

The secrets used to connect each component must not have been migrated over to Vault correctly. You know your team member has already migrated the secret into Vault, but you don't know at what path it is stored.

Everything in Vault is path based, so the first thing you will need to do is find out at what path the kv secrets engine is mounted.
```bash
vault secrets list
```

Notice that the `kv` secrets engine is mounted at the path `kv`/. Next, you need to list the keys of that `kv` mount until you find the Customer Profile Postgres database credentials.
```bash
vault kv list kv
```

Note that there is a key inside the `kv` mount already (db). Continue digging using `vault kv list` to get familiar with path based secrets.
```bash
vault kv list kv/db
vault kv list kv/db/postgres
```

You can also open up the Vault UI tab, type "root" into the Token auth method, and log in to visually browse the kv mount.

Since you know the `products-api` needs to talk to the `postgres` service, you should start there. You can `read` a secret with the read command.
```bash
vault read kv/db/postgres/product-db-creds
```

Something about the password doesn't look right...

Now that you have an idea of how path based secrets are laid out in Vault's `kv` secrets engine, let's start fixing the HashiCups store.

# Migrating Static Database Secrets
The first secret that needs your attention is the credential set for the Products database. These credentials are used by the HashiCups Store's Product API service to pull in all of the products available to consumers.

The credentials are currently stored in a text file on a shared mount. All of your team members at HashiCups have access to this shared mount, and whenever you need to deploy the app, a DBA SSHes into a box, references the file in the shared mount, and updates the app.

When the HashiCups Store was audited, the auditors found, among other things, that the Postgres Products database credentials were clearly visible to all internal team members on a shared drive, at `/share/postgres-product-creds.txt`.

This file is viewable in the "Shared Drive" tab. Noting that what you read in the shared file and what is in Vault do not match, you should make an update to the path in Vault with the right credentials.
```bash
vault kv put kv/db/postgres/product-db-creds \
  username="postgres" \
  password="password"
```

Once you have made the updates, you should redeploy the app so that it pulls the latest credentials.

As noted earlier, you are running everything in Kubernetes, so you will leverage the Vault Agent Sidecar Injector for Kubernetes. The Vault Agent leverages the Kubernetes auth method to authenticate, and then injects secrets in a few ways.

You don't have to worry about how this is set up for now, but you should understand what the containers it can inject do.

Two types of Vault Agent containers can be injected: init and sidecar.

The init container will prepopulate the shared memory volume with the requested secrets prior to the other containers starting. The sidecar container will continue to authenticate and render secrets to the same location as the pod runs. Using annotations, the initialization and sidecar containers may be disabled.

At a minimum, every container in the pod will be configured to mount a shared memory volume. This volume is mounted to /vault/secrets and will be used by the Vault Agent containers for sharing secrets with the other containers in the pod.

You can see where your team specifies the path for the secret if you look inside products-api.yml file, which you can find under the "k8s Dir" tab.

Look for spec -> template -> metadata -> annotations -> vault.hashicorp.com.* to understand what is being read from Vault, and written to the local filesystem.

Look for spec -> template -> metadata -> annotations -> vault.hashicorp.com.* to understand what is being read from Vault, and written to the local filesystem.

Now that you understand how the Vault Agent Sidecar Injector for Kubernetes works, it's time to cycle the deployment and read in the new secrets using the init container. Since we are running all of our components in Kubernetes, we can do that with kubectl.

You can see all of the deployments in your Kubernetes cluster.
```bash
kubectl get deploy
```

You should restart the products-api-deployment, you can do that with a single command.
```bash
kubectl rollout restart deployment products-api-deployment
```

Keep checking the deployment status until it's up and running.
```bash
kubectl get deploy
```

Once the deployment has been restarted, open up the HashiCups Store tab again and refresh the page. You should be able to see one of the HashiCups products now.

Now that you've migrated the credentials from the old shared file into Vault, remove it.
```bash
rm /share/postgres-product-creds.txt
```

Now that the secret is protected by Vault, you've taken one step in the right direction towards good secrets management hygiene.