# Harbor

## Purpose

This directory deploys Harbor as the private container registry for CI image storage and image scanning.

Current deployment:

```text
Helm chart: 1.19.2
Harbor application: 2.15.2
Namespace: harbor
Release: harbor
External URL: https://registry.domain.com
```

## Responsibilities

The Harbor module is responsible for:

- adding the Harbor Helm repository;
- rendering Harbor Helm values;
- configuring ingress/TLS and the external URL;
- provisioning the Harbor core secret with the correct chart key;
- configuring Harbor admin credentials from Vault-backed values;
- configuring registry/database/Redis/Trivy persistence;
- configuring desired component replica counts;
- deploying Harbor Core, Portal, Registry, Jobservice, Redis, database, and Trivy according to the Helm values;
- providing a stable registry endpoint for CI.

## Current storage targets

```text
Registry   10Gi
Database    5Gi
Redis       1Gi
Trivy       5Gi
```

Registry storage was deliberately reduced to `10Gi` in the current working configuration.

## Secret handling

Two important protected values are managed through Vault-backed data:

- Harbor administrator password;
- Harbor core secret.

The Harbor core Kubernetes Secret must use the key expected by the current chart:

```text
secretKey
```

Using a different key such as `secret` causes the chart validation/startup path to fail.

## CI integration model

GitLab CI uses Harbor as an image registry:

```text
GitLab CI
   |
   | docker/OCI push
   v
registry.domain.com
   |
   +--> Harbor project
   +--> Registry storage
   +--> Trivy scanning
```

Robot-account credentials belong in CI variables/secrets, not in `group_vars/all.yml` or normal Git files.

## Prerequisites

Before applying:

1. Kubernetes storage class exists.
2. cert-manager and Cloudflare DNS-01 issuance work.
3. DNS for `registry.domain.com` points to the Cilium ingress address.
4. Vault contains Harbor protected values.
5. The cluster has enough storage for the requested PVC sizes.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/7-harbor/harbor.yml \
  --ask-vault-pass
```

## Validation

```bash
helm list -n harbor
helm status harbor -n harbor
kubectl get pods -n harbor
kubectl get pvc -n harbor
kubectl get ingress -n harbor
```

Check HTTPS:

```bash
curl -I https://registry.domain.com
```

Check the Harbor API health endpoint:

```bash
curl -s https://registry.domain.com/api/v2.0/ping
```

Expected response:

```text
Pong
```

## Functional registry test

From a client with valid Harbor credentials:

```bash
docker login registry.domain.com
```

Then validate a real image push/pull path rather than relying only on the portal returning HTTP 200.

## Troubleshooting

### Harbor Core does not start

Check:

```bash
kubectl get pods -n harbor
kubectl logs -n harbor deploy/harbor-core --tail=200
```

Pay special attention to secret/config validation and dependency reachability.

### Jobservice is restarting

Harbor Jobservice may restart while a dependency such as Core is unavailable. Check both sides:

```bash
kubectl logs -n harbor deploy/harbor-core --tail=200
kubectl logs -n harbor deploy/harbor-jobservice --tail=200
```

### Portal works but image push fails

Portal availability does not prove the Registry path works. Trace:

```text
client
  -> ingress
  -> Harbor Core
  -> Registry
  -> registry storage
```

Also verify authentication/robot-account permissions.

### PVC remains Pending

```bash
kubectl get pvc -n harbor
kubectl describe pvc -n harbor <pvc-name>
```

Check StorageClass and available capacity.

### Trivy is running but scanning is not configured

A healthy Trivy pod does not by itself prove every desired Harbor scan policy is enabled. Validate the actual Harbor project/scanning configuration through Harbor.

## Security

- Never commit Harbor admin passwords.
- Never commit robot-account tokens.
- Do not expose registry service ports directly when ingress is the intended entry point.
- Keep Harbor persistence enabled for stateful data.

## Upgrade procedure

1. Review Harbor chart compatibility.
2. Verify storage and database compatibility.
3. Update the chart version in `group_vars/all.yml`.
4. Run syntax-check.
5. Apply the playbook.
6. Watch all Harbor components.
7. Test `/api/v2.0/ping`.
8. Test login and push/pull.
9. Verify scanning.

## Suggested commit

```text
feat(harbor): document registry deployment storage and CI integration
```
