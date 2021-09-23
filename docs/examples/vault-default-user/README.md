# HashiCorp Vault Default User Example

The RabbitMQ Cluster Operator supports storing RabbitMQ default user (admin) credentials and RabbitMQ server certificates in
[HashiCorp Vault](https://www.vaultproject.io/).

As explained in [this KubeCon talk](https://youtu.be/w0k7MI6sCJg?t=177) there four different approaches in Kubernetes to consume external secrets:
1. Direct API
2. Controller to mirrors secrets in K8s
3. Sidecar + MutatingWebhookConfiguration
4. Secrets Store CSI Driver

In this example, we take the 3rd approach (`Sidecar + MutatingWebhookConfiguration`) integrating with Vault using [vault-k8s](https://github.com/hashicorp/vault-k8s). If `spec.secretBackend.vault.pathDefaultUser` is set in the RabbimqCluster CRD, the Cluster Operator will **not** create a K8s Secret for the default user credentials. Instead, Vault init and sidecar containers will fetch username and password from Vault.

If `spec.secretBackend.vault.tls.pathCertificate` is set, short-lived server certificates are issued from [Vault PKI Secrets Engine](https://www.vaultproject.io/docs/secrets/pki) upon every RabbitMQ Pod (re)start. See [examples/vault-tls](../vault-tls) for more information.

(This Vault integration example is independent of and not to be confused with the [Vault RabbitMQ Secrets Engine](https://www.vaultproject.io/docs/secrets/rabbitmq).)

## Usage

This example requires:
1. Vault server is installed.
2. [Vault agent injector](https://www.vaultproject.io/docs/platform/k8s/injector) is installed.
3. [Vault Kubernetes Auth Method](https://www.vaultproject.io/docs/auth/kubernetes) is enabled.
4. The RabbitMQ admin credentials were already written to Vault to path `spec.secretBackend.vault.pathDefaultUser` with keys `username` and `password` (by some cluster-operator external mechanism. The cluster-operator will never write admin credentials to Vault).
5. Role `spec.secretBackend.vault.role` is configured in Vault with a policy to read from `pathDefaultUser`.

Run script [setup.sh](./setup.sh) to get started with a Vault server in [dev mode](https://www.vaultproject.io/docs/concepts/dev-server) fullfilling above requirements. (This script is not production-ready. It is only meant to get you started experiencing end-to-end how RabbitMQ integrates with Vault.)

You can deploy this example like this:

```shell
kubectl apply -f rabbitmq.yaml
```

And once deployed, you can check check that the admin user credentials got provisioned by Vault:

```shell
kubectl exec vault-default-user-server-0 -c rabbitmq -- rabbitmqctl authenticate_user <username> <password>
```
where `<username>` and `<password>` are the values from step 4 above.

## Admin password rotation without restarting Pods
Rotating the admin password (but not the username!) is supported without the need to restart RabbitMQ servers.

This is how it works:
1. The RabbitMQ cluster operator deploys a sidecar container called `rabbitmq-admin-password-updater`
2. When the default user password changes in Vault, the Vault sidecar container updates the new admin password from Vault server into file `/etc/rabbitmq/conf.d/11-default_user.conf`
3. The `rabbitmq-admin-password-updater` sidecar monitors the file `/etc/rabbitmq/conf.d/11-default_user.conf` and when it changes, it updates the password in RabbitMQ.
4. Additionally, the sidecar updates the local file `/var/lib/rabbitmq/.rabbitmqadmin.conf` with the new password (required by the local rabbitmqadmin tool)

Although we do not need to set the `rabbitmq-admin-password-updater` image name, we can override it like shown below
```
   vault:
      role: rabbitmq
      defaultUserPath: secret/data/rabbitmq/config
      defaultUserUpdaterImage: "rabbitmqoperator/admin-password-updater:1.0.1"
```

To disable the sidecar container, set the image name to an empty string as shown below
```
   vault:
      role: rabbitmq
      defaultUserPath: secret/data/rabbitmq/config
      defaultUserUpdaterImage: ""
```