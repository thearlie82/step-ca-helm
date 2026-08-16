# step-ca-helm

Umbrella Helm chart that wraps upstream [`smallstep/step-certificates`](https://github.com/smallstep/helm-charts/tree/master/step-certificates)
(the step-ca certificate authority) and adds OpenShift-specific resources:

- OpenShift **passthrough** `Route` for the CA API (step-ca serves its own TLS on `:9000`)

Per-environment values live in [`adt-grc-gitops`](https://github.com/thearlie82/adt-grc-gitops)
under `environments/step-ca/<env>/values.yaml`.

## Why passthrough (not edge)

step-ca *is* a CA — it presents a leaf cert signed by its own root/intermediate.
An edge or reencrypt route would make clients trust the router's cert instead of
the step-ca root, breaking `step ca bootstrap` and ACME validation. The router
therefore forwards raw TLS straight to the pod.

## Prereqs in the cluster

1. Namespace `step-ca` created (`oc new-project step-ca`)
2. A default StorageClass (the CA database uses a 1Gi PVC by default)

## Local render

```bash
helm dependency update
helm template step-ca . -f values.yaml
```

> Install as release name **`step-ca`** — the passthrough Route points at a
> Service named `step-ca`, and the upstream subchart names its Service after
> the release.

## Test CA vs real CA

Defaults here are for a **throwaway test CA**: bootstrap auto-generates the root
CA password and provisioner password into Secrets. For anything real, supply
`step-certificates.ca.password` / `step-certificates.ca.provisioner.password`
out of band (never in git) and back the database with durable storage.

## After deploy — trust the CA / issue a cert

```bash
# Root fingerprint is printed by the bootstrap job and stored in the
# step-ca-config ConfigMap / step-ca-secrets Secret.
oc get secret step-ca-ca-password -n step-ca -o jsonpath='{.data.password}' | base64 -d

# Bootstrap a client against the CA (fingerprint from the bootstrap job logs):
step ca bootstrap \
  --ca-url https://step-ca.apps.cluster.adt.network \
  --fingerprint <root-fingerprint>

# Health check
curl --cacert $(step path)/certs/root_ca.crt \
  https://step-ca.apps.cluster.adt.network/health   # -> {"status":"ok"}
```
