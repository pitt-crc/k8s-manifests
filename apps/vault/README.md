# Vault

HashiCorp Vault is a secrets management tool for securely managing sensitive values.
This deployment runs Vault as a multi-node high availability cluster.

## Unsealing

Vault uses a security mechanism called **sealing**. When a Vault pod starts or restarts, it begins in a locked
(*sealed*) state where it cannot serve any requests. To make it operational, an operator must provide a cryptographic
key to *unseal* it. This ensures Vault's contents cannot be accessed without explicit human involvement after a restart.

This deployment uses **Secret Sharing**, which requires multiple keys to be provided together to unseal a node.
No single person needs to hold all the keys - shares can be distributed across team members so that no individual can
unseal the Vault alone.

### How to Unseal

Export your unseal keys as environment variables. The following example assumes Vault is configured to require three
keys.

```bash
export UNSEAL_KEY_1="..."
export UNSEAL_KEY_2="..."
export UNSEAL_KEY_3="..."
```

Then unseal all nodes in one pass:

```bash
for pod in vault-0 vault-1 vault-2 vault-3 vault-4; do
  echo "Unsealing $pod"
  echo "$UNSEAL_KEY_1" | kubectl exec -i $pod -n admin-vault -- vault operator unseal -
  echo "$UNSEAL_KEY_2" | kubectl exec -i $pod -n admin-vault -- vault operator unseal -
  echo "$UNSEAL_KEY_3" | kubectl exec -i $pod -n admin-vault -- vault operator unseal -
done
```

Verify each pod in the deployment is healthy after unsealing:

```bash
kubectl exec -it vault-0 -n admin-vault -- vault status
```

A healthy node will show `Sealed: false` and `HA Mode: active` or `standby`.

### Important Notes

- Unseal the `vault-0` pod first. It is typically elected as the *lead* node in HA mode and the other nodes need it to
  be reachable to join the cluster.
- Pods that remain sealed too long will be automatically restarted by Kubernetes. Administrators may wish to restart
  the sealed pod(s) manually before unsealing to maximize the time they have while executing the unseal commands.
- If a node re-seals itself after being unsealed, it means it lost contact with the active node and needs to be  
  unsealed again.
