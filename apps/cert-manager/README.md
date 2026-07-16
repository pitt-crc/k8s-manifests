# cert-manager

cert-manager is a Kubernetes controller that issues and renews TLS
certificates automatically. This deployment installs cert-manager into
the cluster and configures a Sectigo certificate authority other 
components can use to obtain publicly trusted certificates.

## What Is Deployed

The manifests in this directory deploy cert-manager and its supporting
resources into the `admin-cert-manager` namespace. cert-manager itself
is installed from the official Jetstack Helm chart.

cert-manager works declaratively. Once running, its controller watches
for `Certificate` and `Issuer` resources anywhere in the cluster and
reconciles reality to match them. When a `Certificate` asks for a
hostname that has no valid certificate yet, the controller requests one
from the referenced issuer, completes whatever challenge the issuer
demands, and writes the signed certificate into a Kubernetes secret. It
then watches the expiry date and repeats the process before the
certificate lapses, so certificates renew without operator involvement.

To make sure only one copy of its cert-manager controller is active 
at a time, cert-manager uses a shared marker called a lease. Cert-manager
defaults to storing this lease in the `kube-system` namespace, which is
off limits to apps deployed into the CRCD cluster. Instead, the lease is
moved into cert-manager's own namespace, where it can be managed without 
trouble (see relevant settings in `values.yaml`).

## Issuing Public Certificates

The `sectigo-issuer` `ClusterIssuer` is the only certificate authority
referenced by this deployment. Being a *cluster* issuer rather than a
namespaced one, it can sign certificates for `Certificate` resources in
any namespace, which is why a single definition here serves the whole
cluster.

It issues certificates through ACME, the same automated protocol used by
Let's Encrypt, but pointed at Sectigo's enterprise ACME service rather
than a free public CA. Sectigo requires that each ACME account be tied
to a paid enterprise account before it will issue anything, a step known
as *External Account Binding*. The binding is proven with a key
identifier and a matching HMAC secret; cert-manager presents both when
it first registers its ACME account, and Sectigo uses them to associate
the new account with the organisation's Sectigo subscription. The key
identifier is recorded inline in the issuer, while the HMAC secret is
supplied separately (see below). Once registration succeeds,
cert-manager stores the resulting ACME account key in its own secret and
reuses it for every subsequent request.

## Supplying EAB Credentials from Vault

The External Account Binding HMAC secret is a long-lived credential
held in Vault and delivered into the cluster at runtime by the Vault
Secrets Operator. Three resources arrange this synchronisation:

- A `VaultConnection` records the address of the Vault server.
- A `VaultAuth` configures which service account to use when authenticating against Vault
- A `VaultStaticSecret` names the exact secret to pull

### Important Notes

- This issuer depends on Vault and the Vault Secrets Operator. If either
  is unavailable the `sectigo-eab-credentials` secret will not be
  created, and the `ClusterIssuer` cannot register with Sectigo.
- The Vault key/value entry backing the EAB binding must contain the
  HMAC secret under the key the issuer expects (`eab_hmac`). If the key
  name in Vault does not match, the synced secret will exist but the
  issuer will fail to register.
- The EAB credentials are re-read from Vault every hour. A credential
  rotated in Vault is not picked up instantly; expect up to an hour of
  lag before the cluster secret reflects the change.
