# Epinio catalog blueprint

[`../dev/epinio-0.1.0.yaml`](../dev/epinio-0.1.0.yaml) is a portable AIF
template for the official Epinio chart **1.14.2**, app **v1.14.2**. Import it,
**Copy** it to a local Blueprint, configure that copy, then install it. The
published template deliberately has no domain or registry password.

The chart was downloaded from the [official repository](https://epinio.github.io/helm-charts)
and checked against its index. Archive SHA-256:
`6a714a47856457cce130b31c4002e01ca5b3ef51bd63aae47ceda69e5cc6cdb4`.
See the [release](https://github.com/epinio/helm-charts/releases/tag/epinio-1.14.2)
and [installation documentation](https://docs.epinio.io/installation/install_epinio).

## Before installation

- Register the public Helm source on the AIF management cluster:
  `kubectl apply -f blueprints/epinio/cluster-repo.yaml` from your repository root.
  This is separate from the Blueprint catalog: the importer accepts Blueprint
  documents only, and `chartRepo: epinio` refers to this ClusterRepo's name.
- The target cluster needs an existing ingress controller, cert-manager, and a
  default StorageClass. This template requests RWO PVCs: 10Gi for the image
  registry, 10Gi for source volumes, 1Gi each for source metadata and master
  data, and 2Gi for image exports; staging caches request additional storage.
  It uses the default provisioner rather than assuming local-path or a CSI vendor.
- Wildcard DNS for `*.<domain>` must reach the ingress controller from clients
  and the cluster. `sslip.io` is one option; an organization-owned domain works too.
- Use one Epinio installation per cluster. Upstream creates shared ClusterIssuers
  (`selfsigned-issuer`, `epinio-ca`), a `workspace` namespace, and registry NodePort
  **30500**, even with a custom installation namespace. Check for name/port
  conflicts first; AIF's deployment path permits Helm to take resource ownership.
- See upstream [system requirements](https://docs.epinio.io/getting-started/system-requirements)
  and [cluster prerequisites](https://docs.epinio.io/how-to/operator/cluster-prerequisites).
  Installing Epinio also installs its packaged SeaweedFS, Reflector and application
  Helm controller. These are upstream chart dependencies, not additions to AIF.

## Configure the imported template

In AIF, refresh your Dev catalog, select Epinio, choose **Copy**, and give the
copy a local name such as `Epinio Lab`. In **Configuration**, expand `epinio`
and select **YAML**. Edit the values below while preserving the other template
settings, then review and create the copy. Do not select **Load defaults**:
it replaces these settings with upstream defaults. The upstream Form hides
the bundled registry's password and shows default-user fields this template disables.

| Value | Local Rancher example | Another installation |
| --- | --- | --- |
| `global.domain` | `192.168.2.100.sslip.io` | `apps.example.com` |
| `ingress.ingressClassName` | `traefik` | Your ingress class, or empty for the default |
| `global.registryPassword` | A unique, privately generated password | A unique, privately generated password |
| `global.tlsIssuer` | `epinio-ca` | `epinio-ca`, or an upstream supported issuer |
| `global.customTlsIssuer` | Empty | An existing **ClusterIssuer** name if using managed TLS |
| `certManagerNamespace` | `cert-manager` | Your cert-manager namespace |

For example, generate a registry password with `openssl rand -hex 32` and put
it only in the local copy. Never publish that configured copy to your catalog.
The upstream registry template consumes this password as a Helm value, and
current AIF does not support a Secret reference for it. Anyone permitted to read
the local Blueprint or generated Helm values can read it. Environments requiring
Secret-only credential storage need that integration addressed before deployment.

The empty domain fails chart validation; the null registry password also stops
rendering after a domain is supplied. Replace the null with your unique password,
not an empty string. These guards prevent inheriting upstream's example password.

Current AIF's install wizard does not collect per-install Helm values. Although
`AIWorkload.spec.componentValues` exists in the schema, Blueprint reconciliation
does not consume it. Configure the local Blueprint copy instead. Imported versions
stay immutable and portable; publish a new template version for later changes.

Choose `epinio` as the target namespace when installing the copy. A different
namespace works if you create the user Secret below in that namespace too.

## Create the administrator in a Secret

This template disables the default API accounts and Dex. In this chart version,
Dex also creates accounts with known passwords even if `api.users` is empty.
Epinio's standalone UI and CLI support local authentication without Dex.

Before installing, create a user Secret on the **target cluster**. The following
uses Apache's `htpasswd` to prompt for a password and produce a bcrypt hash;
the plaintext password is not written to the command line or catalog. Run in Bash
with the target cluster selected in kubectl:

```bash
(
  set -euo pipefail
  epinio_auth_dir=$(mktemp -d)
  trap 'rm -rf "$epinio_auth_dir"' EXIT
  printf '%s' admin > "$epinio_auth_dir/username"
  htpasswd -nBC 12 admin | cut -d: -f2 | tr -d '\n' > "$epinio_auth_dir/password"
  kubectl create namespace epinio --dry-run=client -o yaml | kubectl apply -f -
  kubectl -n epinio create secret generic epinio-admin --type=BasicAuth \
    --from-file=username="$epinio_auth_dir/username" \
    --from-file=password="$epinio_auth_dir/password" \
    --dry-run=client -o yaml | kubectl apply -f -
  kubectl -n epinio label secret epinio-admin \
    epinio.io/api-user-credentials=true --overwrite
  kubectl -n epinio annotate secret epinio-admin epinio.io/roles=admin --overwrite
)
```

Keep this Secret separate from the Blueprint and public repository. See upstream
[authorization](https://docs.epinio.io/next/references/authorization) for roles
and user management. Use a different password from the registry password.

## Ingress, certificates, and runtime limits

With the local values above, the Epinio UI/API is
`https://epinio.192.168.2.100.sslip.io`. Rancher keeps its own hostname; both
use the same Traefik controller and address. Epinio applications also inherit
the selected ingress class. For another domain, the endpoint is `https://epinio.<domain>`.

Reusing ingress does not reuse Rancher's certificate. The default `epinio-ca`
creates an Epinio CA and certificates through the existing cert-manager. Trust
that CA on your client before using the endpoint; do not disable certificate
verification globally. Its public certificate can be exported after issuance:

```bash
kubectl -n cert-manager get secret epinio-ca-root \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > epinio-ca.crt
```

For trusted public DNS, set `global.customTlsIssuer` to an existing ClusterIssuer,
or use upstream's `letsencrypt-production` and set `global.tlsIssuerEmail`.
HTTP-01 issuance requires a reachable domain; the private local address is not
an ACME public-ingress example. Rancher's namespaced Issuer is not a ClusterIssuer.

The bundled registry uses HTTP `127.0.0.1:30500` for node image pulls. The runtime
must allow this endpoint; it is separate from the local development registry on
port 5000. Upstream supports an external registry through
`containerregistry.enabled: false` and `global.registry*` values when encrypted
node pulls or an existing registry are required. Reflector copies `registry-creds`
into application namespaces; upstream currently enables reflection into all
namespaces. This is a material permissions consideration on shared clusters.

Persistent claims replace upstream hostPath/temporary data defaults, but these
single-replica services are not highly available. The local example's provisioner
uses node-local storage. Choose storage, capacity, backups and registry policy for
the target environment. Chart rendering does not verify node image pulls or builds.

## Publish and verify

Keep `cluster-repo.yaml` and this README outside `dev/blueprints.yaml`. After
editing authored files, regenerate from the directory containing `dev/`:

```bash
(
  set -euo pipefail
  export LC_ALL=C
  {
    printf '# Generated from individual Blueprint files in dev/.\n'
    for blueprint_file in dev/*.yaml; do
      [ "$blueprint_file" = dev/blueprints.yaml ] && continue
      printf '\n---\n'
      cat "$blueprint_file"
    done
  } > dev/blueprints.yaml.tmp
  mv dev/blueprints.yaml.tmp dev/blueprints.yaml
)
```

The Dev catalog contains the nine existing versions plus this template. Prod
has not been changed. Publish when ready, refresh the saved catalog, and configure
a local copy. This work does not push to GitHub or install Epinio.

Validation covers the official chart digest, local Traefik/sslip.io rendering,
another domain/ingress class, no default user accounts, persistent data claims,
required configuration guards, and Kubernetes admission of the Blueprint and
chart repository. A live install, login, app build/push and persistence test
remain necessary before promoting the template to Prod.
