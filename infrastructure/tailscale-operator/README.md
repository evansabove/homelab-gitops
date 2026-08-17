# Tailscale operator credentials

The operator authenticates to the tailnet with an OAuth client, which it reads
from a Secret named `operator-oauth` in the `tailscale` namespace. That Secret
is **not** committed in plaintext — generate it yourself with `kubeseal` so the
credentials never leave your machine unencrypted, then commit the result here.

## 1. Tailnet prerequisites (admin console)

Add the operator's tags to the tailnet policy file, under `tagOwners`:

```
"tagOwners": {
    "tag:k8s-operator": [],
    "tag:k8s":          ["tag:k8s-operator"],
}
```

Then, under DNS, enable **MagicDNS** and **HTTPS Certificates** — the Tailscale
ingress terminates TLS with a tailnet-issued cert and needs both.

Finally create an OAuth client (Settings > OAuth clients) with write access to
Auth Keys, tagged `tag:k8s-operator`. Copy the client ID and secret.

## 2. Seal the credentials

Run this **from WSL** (kubeseal lives at `~/.local/bin` there, not on Windows),
from `/mnt/c/Projects/homelab-gitops`, substituting the two values:

```bash
kubectl create secret generic operator-oauth \
  --namespace tailscale \
  --from-literal=client_id=<CLIENT_ID> \
  --from-literal=client_secret=<CLIENT_SECRET> \
  --dry-run=client -o yaml \
| kubeseal --context homelab --format yaml \
  > infrastructure/tailscale-operator/operator-oauth.yaml
```

`--context homelab` is not optional. This machine's default kubectl context is
a work cluster, and kubeseal encrypts against whichever cluster it is pointed
at — seal against the wrong one and the homelab controller cannot decrypt it.
The `kubectl create secret` half runs entirely offline (`--dry-run=client`), so
the plaintext is never sent anywhere; it is piped straight into kubeseal.

Commit `operator-oauth.yaml` and push. ArgoCD applies it, the sealed-secrets
controller decrypts it into the real `operator-oauth` Secret, and the operator
starts.

Note kubeseal encrypts against the cluster's public key, so the output is safe
to commit but only decryptable by this cluster.
