# ad-argocd-apps

ArgoCD application definitions for the `ad-accel-online-ml` (prod) and `ad-accel-online-ml-dev`
(dev) vclusters at SLAC SDF.

| Cluster | ArgoCD UI (log in with **SLAC SSO**) | Root app file | Watches |
|---|---|---|---|
| prod | https://ad-accel-online-ml.slac.stanford.edu/argo-cd | `root-app.yaml` | `apps/` |
| dev | https://ad-accel-online-ml-dev.slac.stanford.edu/argo-cd | `root-app-dev.yaml` | `apps-dev/` |

## How it works

Each cluster runs an `apps-root` "app of apps" that watches its folder on `main`. Each file in
that folder is an ArgoCD `Application` (or `AppProject`) pointing at a deployment repo.

**Prod** (`apps/`):

| Application | Project | Deploys from | Namespace |
|---|---|---|---|
| `inference-services` | `lume-online-ml` | `slaclab/inference-service` (`deployments/`) | `lume-online-ml` |
| `lcls-cu-inj-model-deployment` | `lume-online-ml` | `slaclab/lcls-cu-inj-model-deployment` | `lume-online-ml` |
| `lcls-fel-surrogate-deployment` | `lume-online-ml` | `slaclab/lcls-fel-surrogate-deployment` | `lume-online-ml` |
| `lume-visual-dashboard` | `lume-visual-dashboard` | `slaclab/lume-visual-dashboard` (`deploy/kubernetes/overlays/prod`) | `lume-visual-dashboard` |

**Dev** (`apps-dev/`):

| Application | Project | Deploys from | Namespace |
|---|---|---|---|
| `lume-visual-dashboard` | `lume-visual-dashboard` | `slaclab/lume-visual-dashboard` (`deploy/kubernetes/overlays/dev`) | `lume-visual-dashboard` |

Dev also runs `slacathon26`, which was created by hand and isn't defined in this repo.

There are two places to make changes:

- **To change what is deployed** (images, replicas, config), edit the app's own deployment repo.
  ArgoCD picks up the change on `main` (or the app's `targetRevision`) within ~3 minutes.
- **To change how an app is deployed** (source repo, branch, path, namespace, project, sync
  options), edit its file in `apps/` or `apps-dev/` in this repo.
- **To change `apps-root` itself**, edit `root-app.yaml` (prod) or `root-app-dev.yaml` (dev) and
  apply it by hand after merging, with the matching kubectl context:
  `kubectl --context ad-accel-online-ml apply -f root-app.yaml` or
  `kubectl --context ad-accel-online-ml-dev apply -f root-app-dev.yaml`. Nothing syncs these files
  automatically, so keep the live apps and the files in step.

## Permissions

There are two separate permission systems. They are **not** linked.

**ArgoCD UI** (defined in the `argocd-rbac-cm` ConfigMap on the cluster, not in this repo):

- **Admins** have full access to all apps and projects.
- **Project developer roles** let you view, sync, restart, roll back, and view logs for apps in
  one project only. You can't change the project itself (allowed repos/namespaces).
- **Everyone else** who logs in with SSO gets no access by default.

| Project | Cluster | Allowed source repo | Allowed namespace | Developer role |
|---|---|---|---|---|
| `default` | both | any | any | admins only (used by `apps-root`; on dev also `slacathon26`) |
| `lume-online-ml` | prod | `slaclab/inference-service`, `slaclab/lcls-cu-inj-model-deployment`, `slaclab/lcls-fel-surrogate-deployment` | `lume-online-ml` | admins only |
| `lume-visual-dashboard` | prod, dev | `slaclab/lume-visual-dashboard` | `lume-visual-dashboard` | `role:lume-visual-dashboard-developer` |
| `badger-gui` | prod | `xopt-org/badger` | `badger-gui` | `role:badger-gui-developer` |

Project definitions (except `default`) live in `apps/appproject-*.yaml` and
`apps-dev/appproject-*.yaml` and are synced by `apps-root` like the Applications.
Each cluster has its own ArgoCD, so roles and admins are set separately on each. Changing them needs code-owner approval.

**kubectl** access comes from Kubernetes RBAC on the vcluster (`vcluster-admins` and
`cluster-admin` ClusterRoleBindings). Being in either list gives full cluster-admin, separate
from your ArgoCD role.

**This repo**: `main` requires a pull request approved by a code owner (see
`.github/CODEOWNERS`). You can't approve your own PR.

## For developers

**Adding a new app**

1. Put your Kubernetes manifests (plain YAML or Kustomize) in your app's repo.
2. Ask an admin for a dedicated ArgoCD project scoped to your repo and namespace, and a
   developer role for that project. New apps should not go in `default`.
3. Open a PR here adding `apps/<your-app>.yaml`, modeled on `apps/lume-visual-dashboard.yaml`,
   with `project:` set to your project. Add yourself to `.github/CODEOWNERS` for that file.
4. Once merged, `apps-root` creates the Application and it starts syncing.

**Things to know**

- **Changes made in the UI aren't permanent.** `apps-root` doesn't self-heal, so an app edited in
  the UI keeps your changes until the next push to this repo. Then it reverts to what's in
  `apps/`. Put lasting changes in git.
- **Deleting a file from `apps/` doesn't delete the app.** `apps-root` doesn't prune, so remove
  the Application in the UI as well (or ask an admin).
- **Deleting an app in the UI can delete its running resources.** If the app is still in
  `apps/`, it gets recreated on the next sync of `apps-root`.
- **Apps sync automatically but don't self-heal.** Manual `kubectl` edits to resources an app
  manages are flagged as OutOfSync and are not reverted until the next sync.
- **Projects limit what your app can deploy.** If your deployment repo adds a cluster-wide
  resource (e.g. a ClusterRole, CRD, or Namespace) or targets a namespace outside its project,
  the app's sync fails with a "not permitted in project" error. Widening a project needs a PR
  here approved by a code owner (see `.github/CODEOWNERS`).
- **Adding a new repo or app to an existing project** also means adding the repo to that
  project's `sourceRepos` in `apps/appproject-<project>.yaml`.
- **The UI pod terminal is disabled.** Use `kubectl exec` if you have kubectl access.
- **If an app shows `Unknown` sync status**, the source usually can't be resolved, e.g. its
  `targetRevision` branch was deleted. Check the app's conditions in the UI.

## Getting access

Ask an ArgoCD admin to add you to the right project developer role. For kubectl access to
the vcluster, ask a vcluster admin.
