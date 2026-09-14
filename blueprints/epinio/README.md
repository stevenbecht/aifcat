# Epinio local demo

[`../dev/epinio-0.1.1.yaml`](../dev/epinio-0.1.1.yaml) installs the official
Epinio Helm chart **1.14.2** through AIF. Its HTTPS chart repository URL and
example values are contained in one Blueprint. It needs AIF's direct HTTPS
Blueprint chart-source support; no separate ClusterRepo is required.

- Address: `https://epinio.192.168.2.100.sslip.io`
- Administrator: `admin` / `Epinio-Demo-2026!`
- Registry password: `Epinio-Registry-Demo-2026!`

These are public demo credentials. Replace both passwords before using the
example outside the local lab. Blueprint and Helm values contain the passwords;
the chart creates the administrator and registry Secrets from those values.
Dex is disabled and only the specified administrator account is created.

## Demo: add a catalog and install Epinio

The target cluster needs Traefik, cert-manager in namespace `cert-manager`,
a default StorageClass, and DNS/ingress access to `*.192.168.2.100.sslip.io`.
Install only once per cluster: the chart creates shared ClusterIssuers,
a `workspace` namespace, and registry NodePort 30500.

Before presenting, publish `blueprints/dev/epinio-0.1.1.yaml` and update
`blueprints/dev/blueprints.yaml` in the GitHub repository. The existing index
already lists the other Blueprint files; keep those references and files.
The Epinio entry should be:

```yaml
resources:
  - epinio-0.1.1.yaml
```

Prepare the demo cluster with no Epinio installation and without this catalog
registration. On the current lab, uninstall `epinio-local` through AIF and wait
for Helm teardown before another installation. Remove the saved Partner
Blueprints catalog before demonstrating Add. Removing a catalog retains its
imports, so remove unused Epinio definitions as well if the demo should show
Epinio appearing for the first time. Retained PVCs and certificates need review
when preparing a completely fresh installation; catalog removal is not a reset
of application data.

During the demonstration:

1. Open **Settings → Blueprint catalogs → Add another catalog**.
2. Enter these values:

   | Field | Value |
   | --- | --- |
   | Name | Partner Blueprints |
   | Repository | `https://github.com/stevenbecht/aifcat` |
   | Ref | `master` |
   | Index path | `blueprints/dev/blueprints.yaml` |

3. Click **Save settings**. Saving the new catalog initiates its import.
   Use **Check import status** to read the result; there is no polling timer.
4. Open **Blueprints**, filter by **Partner Blueprints**, and select
   **Epinio 0.1.1** from that catalog.
5. Click **Install**, use workload name `epinio`, namespace `epinio`, target
   cluster `local`, and deployment strategy **Fleet Bundle**. Review and install.
6. Show the workload becoming **Running**, then open
   `https://epinio.192.168.2.100.sslip.io` and log in as
   `admin` / `Epinio-Demo-2026!`.

The catalog imports the Blueprint. Install creates the AIWorkload, and the
existing Fleet/Helm path installs Epinio using the chart URL and values in that
Blueprint. The UI supplies the catalog-qualified family automatically. This
demo requires no Copy step, separately registered Helm repository, or manual
administrator Secret. Trust the local Epinio CA on the presentation client.

## Direct installation with kubectl

Apply the Blueprint directly:

```sh
kubectl apply -f /src/blueprints/dev/epinio-0.1.1.yaml
```

Then select **Epinio 0.1.1 → Install**, target cluster **local**, namespace
**epinio**. No Copy or additional configuration step is needed.

For installation entirely through kubectl, after applying the Blueprint:

```sh
kubectl create namespace epinio --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f - <<'EOF'
apiVersion: ai-factory.suse.com/v1alpha1
kind: AIWorkload
metadata:
  name: epinio-local
  namespace: epinio
spec:
  displayName: Epinio
  source:
    sourceType: Blueprint
    blueprint:
      name: epinio
      version: 0.1.1
  targetNamespace: epinio
  targetClusters: [local]
  deployStrategy: FleetBundle
EOF
```

The kubectl example references the directly applied local Blueprint family.
Applying a Blueprint registers its definition; an AIWorkload requests
installation. Neither action needs a generator.

## Certificates and validation

The example uses Epinio's local CA through the existing cert-manager. Export
its public certificate and trust it on your client to access the HTTPS endpoint:

```sh
kubectl -n cert-manager get secret epinio-ca-root \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > epinio-ca.crt
```

The bundled registry serves node image pulls through HTTP `127.0.0.1:30500`.
Validate a real application build and image pull on the target runtime.
Persistent registry and source storage use the default provisioner; these
single-replica components are intended for this local demo.

Verified on the local RKE2 cluster: AIF UI installation, administrator login,
source upload/build, registry push, node image pull through `127.0.0.1:30500`,
and the test application's HTTPS response with CA verification. The temporary
application was removed after validation. Existing AIF workloads stayed Running.

The chart comes from the [official Epinio repository](https://epinio.github.io/helm-charts).
Verified chart archive SHA-256:
`6a714a47856457cce130b31c4002e01ca5b3ef51bd63aae47ceda69e5cc6cdb4`.
See the [upstream installation documentation](https://docs.epinio.io/installation/install_epinio).
