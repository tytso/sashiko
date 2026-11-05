# e2fsprogs.sashiko.dev

Kubernetes configuration for the Sashiko instance that reviews `e2fsprogs`
(the `ext2`/`ext3`/`ext4` userspace filesystem utilities and libraries). It
runs in a dedicated `sashiko-e2fsprogs` namespace with dual ingestion:

- **Lore NNTP (`linux-ext4@vger.kernel.org`):** Tracks
  `org.kernel.vger.linux-ext4` on `nntp.lore.kernel.org` (`--track`), using the
  per-list `ingest_filter` in `/app/projects/e2fsprogs/mailing_lists.toml` so
  Linux kernel (`fs/ext4`, `fs/jbd2`) and `xfstests`/`fstests` series are
  ignored at ingestion time while `e2fsprogs` patch series are reviewed and
  replied to on-list (`reply_to = ["author", "list"]`, `positive_review = "all"`,
  `cc = ["tytso@mit.edu"]`).
- **GitHub Pull Requests (`tytso/e2fsprogs`):** Ingests pull requests from
  `https://github.com/tytso/e2fsprogs` via both `/api/webhook/github` and
  periodic Pulls REST API reconciliation (`SASHIKO__FORGE__ENABLED=true`,
  `SASHIKO__FORGE__DISABLE_NNTP=false`, `SASHIKO__FORGE__POST_MODE=live`,
  `SASHIKO__FORGE__REPOS=tytso/e2fsprogs`). Because `[defaults]` in
  `/app/projects/e2fsprogs/mailing_lists.toml` sets `reply_to = []` and
  `positive_review = "none"`, PR reviews post exclusively through GitHub
  without enqueueing outbound emails.
- **Pre-existing Bug Tracking:** Enables `SASHIKO__LINUX_BUG__ENABLED=true` and
  periodic upstream fix checks (`SASHIKO__LINUX_BUG__FIX_CHECK_ENABLED=true`)
  against `origin/master` of
  `https://git.kernel.org/pub/scm/fs/ext2/e2fsprogs.git`.

## Layout

| Path | Contents |
| --- | --- |
| `base/app/sashiko-e2fsprogs-k8s.yaml` | Namespace, KSA, 100Gi PVC, Deployment on port 8080 |
| `base/app/management-tools.yaml` | Data manager pod and nightly backup CronJob |
| `base/routing/` | Service, BackendConfig, ManagedCertificate, Ingress, maintenance page |
| `app/`, `routing/`, `maintenance/` | Overlays that are applied |
| `production/kustomization.yaml.example` | Template for environment-specific IDs |

`production/kustomization.yaml` holds the real project IDs and is ignored by
git. Copy the example and edit it before the first apply.

## Cluster Prerequisites

1. Namespace `sashiko-e2fsprogs` (created by `kubectl apply -k app/`).
2. Secret `sashiko-e2fsprogs-secrets` in `sashiko-e2fsprogs` with:
   - `LLM_API_KEY` (required when using API-key LLM providers)
   - `SASHIKO__SMTP__PASSWORD` (required for `mail.kernel.org` SMTP delivery)
   - `SASHIKO__SERVER__JWT_SECRET` (required for sign-in sessions)
   - `SASHIKO__FORGE__WEBHOOK_SECRET` (required for `/api/webhook/github`)
   - Optional GitHub credentials for posting PR reviews and raising API rate
     limits: `SASHIKO__FORGE__API_TOKEN`, or GitHub App credentials
     (`SASHIKO__FORGE__APP_ID`, `SASHIKO__FORGE__INSTALLATION_ID`,
     `SASHIKO__FORGE__APP_PRIVATE_KEY`).
   - Optional `SASHIKO__SERVER__ACL__SECURITY` (comma-separated security
     addresses with global bug read/comment access).
3. Workload Identity binding for
   `<gcp-project>.svc.id.goog[sashiko-e2fsprogs/sashiko-e2fsprogs-ksa]` on the
   GCP service account so the backup job can write to Cloud Storage.
4. Reserved global static IP addresses (`lb-ipv4-e2fsprogs` and
   `lb-ipv6-e2fsprogs`, or patched in `production/kustomization.yaml`) with DNS
   `A`/`AAAA` records for `e2fsprogs.sashiko.dev`.

## Deploying

```bash
# 1. Environment-specific IDs (first time only).
cp production/kustomization.yaml.example production/kustomization.yaml
$EDITOR production/kustomization.yaml

# 2. Application: namespace, service account, PVC, Deployment, backups.
kubectl apply -k app/

# 3. Routing: service, health check, certificate, Ingress.
kubectl apply -k routing/
```

The managed certificate takes 15 to 60 minutes to reach `Active`:

```bash
kubectl get managedcertificate -n sashiko-e2fsprogs -w
```

## GitHub Webhook Configuration

Configure the repository webhook on `https://github.com/tytso/e2fsprogs`:

- Payload URL: `https://e2fsprogs.sashiko.dev/api/webhook/github`
- Content type: `application/json`
- Secret: value of `SASHIKO__FORGE__WEBHOOK_SECRET` in
  `sashiko-e2fsprogs-secrets`
- Events: *Pull requests*

## Submitting a Commit or Series Manually

`/api/submit` requires an `Ingest`-capable bearer credential (an admin API
token or the local operator token in `/data/db/.sashiko-local-token`):

```bash
kubectl exec -n sashiko-e2fsprogs deploy/sashiko-e2fsprogs-deployment -- \
  sh -c 'curl -s -X POST http://localhost:8080/api/submit \
    -H "Authorization: Bearer $(tr -d "\n" < /data/db/.sashiko-local-token)" \
    -H "Content-Type: application/json" \
    -d "{\"type\":\"remote\",\"sha\":\"<commit-sha>\",\"repo\":\"https://git.kernel.org/pub/scm/fs/ext2/e2fsprogs.git\"}"'
```

## Maintenance Page

Switch the Service selector between the static maintenance page and the
application:

```bash
kubectl apply -k maintenance/   # serve the maintenance page
kubectl apply -k routing/       # back to the application
```

## Data

- Database: `/data/db/sashiko-e2fsprogs.db` on `sashiko-e2fsprogs-data-pvc`
  (`100Gi`).
- Repository clone: `e2fsprogs-repo` subpath of the same volume, mounted at
  `/app/third_party/e2fsprogs` and refreshed by the entrypoint on every start.
- Worktrees: `emptyDir` at `/mnt/worktrees`, rebuilt from the clone on demand.
- Backups: nightly at `06:00 America/Los_Angeles` into Cloud Storage, staggered
  after `sashiko.dev` (`03:00`), `sashiko-self` (`04:00`), and `sashiko-gcc`
  (`05:00`).

To inspect or restore the database without holding it open in the application:

```bash
kubectl exec -it -n sashiko-e2fsprogs deploy/data-manager -- bash
```
