# Ansible automation portal (GitOps)

Deploys the [Ansible automation portal operator](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/installing_self-service_automation_portal/install-install_the_automation_portal_operator) (Technology Preview) into namespace `self-service-portal`.

Replaces the former Helm chart deployment (`redhat-rhaap-portal`).

| Argo CD Application | Purpose |
|---------------------|---------|
| `automation-portal` | Operator subscription + `AutomationPortal` CR |

## Deployment order

1. Sync `automation-portal` (namespaces, operator subscription, CR)
2. **Approve the initial InstallPlan** in `automation-portal-operator-system` (subscription uses `installPlanApproval: Manual`)
3. Wait for operator CSV `Succeeded` and CRD `automationportals.automationportal.aap.redhat.com`
4. **AAP prerequisites** — OAuth app, API token (install guide)
5. `05-secrets.yml` — cluster-specific credentials (gitignored)
6. Wait for `AutomationPortal` phase `Running`

Copy the secrets example before editing credentials:

```bash
cp 05-secrets.example.yml 05-secrets.yml
```

Uncomment `05-secrets.yml` in `kustomization.yml` when ready, or apply secrets with `oc apply -f 05-secrets.yml`.

## Required secrets (`05-secrets.yml`)

Apply **before** the `AutomationPortal` CR reconciles. The operator validates secrets on each cycle (`SecretsValid` condition).

### `secrets-rhaap-portal`

| Key | Description |
|-----|-------------|
| `aap-host-url` | AAP instance URL (e.g. gateway route for your AAP deployment) |
| `oauth-client-id` | OAuth application client ID |
| `oauth-client-secret` | OAuth application client secret |
| `aap-token` | AAP API token (Write scope) tied to the OAuth app |

### `portal-registry-auth`

Required for OCI plug-in delivery. Use a [Registry Service Account](https://access.redhat.com/RegistryAuthentication) (not your personal Red Hat password). Secret name **must** match `{AutomationPortal.metadata.name}-registry-auth` — the CR is named `portal`, so the secret is `portal-registry-auth`.

Generate `auth.json` from your RSA, then either paste into `05-secrets.yml` or create directly:

```bash
oc create secret generic portal-registry-auth -n self-service-portal \
  --from-file=auth.json=./auth.json
```

### `secrets-scm` (optional)

Only if you enable GitHub/GitLab SCM integration on the `AutomationPortal` CR.

## Configure the AutomationPortal CR

`06-automationportal.yml` is managed in git and synced by Argo CD. Key fields:

1. `spec.backstage.route.host` — pinned to the existing portal route (update per cluster)
2. `spec.aap.checkSSL: false` — lab/playground only; AAP uses the default OpenShift router cert
3. `spec.permissions` — RBAC admin/super users (mirrors former Helm `values.yaml`)
4. `spec.database.enableLocalDb: true` — operator-managed PostgreSQL (no manual postgres secret)

For all CR fields see the [configuration reference](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/installing_self-service_automation_portal/install-automation_portal_operator_configuration_reference).

### OCI plugin init (~3 min per new pod)

The operator downloads OCI plugins on pod start (~3 min). Avoid stacked rollouts while plugins initialize.

### AAP OAuth login fails (`fetch failed` on `/o/token/`)

The portal backend POSTs to `${AAP_HOST_URL}/o/token/` during login. On clusters using the **default self-signed OpenShift router certificate**, set `spec.aap.checkSSL: false` (lab/playground only). Production should use `spec.backstage.caCertificates.secretRef` instead.

Also confirm the AAP OAuth app **Redirect URI** matches the portal route:

`https://redhat-rhaap-portal-self-service-portal.apps.ocp001.lennysh.net/api/auth/rhaap/handler/frame`

## After deployment

1. Open the portal route and confirm OAuth **Redirect URI** is  
   `https://<portal-host>/api/auth/rhaap/handler/frame`
2. Configure portal RBAC (Administration → RBAC) per the install guide.

Monitor status:

```bash
oc get automationportal portal -n self-service-portal -w
oc get automationportal portal -n self-service-portal \
  -o jsonpath='{.status.phase}{" "}{.status.url}{"\n"}'
```

## Cutover from Helm (`redhat-rhaap-portal`)

There is no in-place Helm→operator migration. Cut over cleanly:

1. **Prepare secrets** — reuse `secrets-rhaap-portal` and `secrets-scm`; copy `auth.json` from `redhat-rhaap-portal-dynamic-plugins-registry-auth` into new secret `portal-registry-auth`
2. **Remove Helm Argo apps** — delete `self-service-portal` and `self-service-portal-prereqs` (removed from git; `cluster-config` prunes them)
3. **Sync `automation-portal`** — approve InstallPlan, wait for operator CSV
4. **Apply secrets** then let `AutomationPortal` CR reconcile
5. **Verify** operator portal is `Running` and route responds
6. **Remove Helm leftovers** in `self-service-portal`:

```bash
oc delete deploy,sts,svc,route -l app.kubernetes.io/instance=redhat-rhaap-portal -n self-service-portal
oc delete pvc -l app.kubernetes.io/instance=redhat-rhaap-portal -n self-service-portal
oc delete secret redhat-rhaap-portal-postgresql redhat-rhaap-portal-dynamic-plugins-registry-auth -n self-service-portal
```

Operator-managed Postgres starts fresh — portal catalog re-syncs from AAP; local Backstage state from the Helm deployment is not migrated.

## Technology Preview

The automation portal operator is Technology Preview only. Red Hat does not recommend it for production workloads yet.

## Public git repository

Keep `05-secrets.yml` out of git (see `.gitignore`). The public repo ships `05-secrets.example.yml` for structure; apply real credentials with `oc apply -f 05-secrets.yml` before the CR reaches `Running`.
