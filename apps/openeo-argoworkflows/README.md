# openeo-argoworkflows

ArgoCD Application for the openEO Argo Workflows API. Deployed per environment
via the kustomize overlays (`overlays/dev` is active in `apps/kustomization.yaml`;
`overlays/stable` is ready for when the stable node comes online).

## Layout

```
base/
  helm-openeo-argoworkflows.yaml   multi-source Application (chart + $values)
  ingress-openeo.yaml
  kustomization.yaml
values/                            ALL Helm values live here (read via $values)
  base.yaml                        common, env-neutral values (no image)
  dev.yaml / stable.yaml           per-env overrides (deep-merged over base)
  image-base.yaml                  API image, env-neutral default
  image-dev.yaml                   API image for dev   (Image Updater writes here)
  image-stable.yaml                API image for stable
overlays/
  dev/                             name -dev, ns openeo-dev, Image Updater on
  stable/                          name openeo, ns openeo, Image Updater off
```

## Why multi-source, and where config lives

The Helm **chart** lives in the `charts` repo, but **all values** live in *this*
repo so they're plain reviewable YAML and so ArgoCD Image Updater can commit image
bumps back here. The Application therefore has two sources:

1. the chart (`charts` repo, path `eodc/openeo-argo`), and
2. this repo as a values-only source (`ref: values`).

`helm.valueFiles` references `$values/apps/openeo-argoworkflows/values/*.yaml`,
merged **in order, later wins**:

```
base.yaml  ->  <env>.yaml (dev/stable)  ->  image-<env>.yaml
```

> ⚠️ There is no inline `valuesObject` on the Application — in ArgoCD it outranks
> `valueFiles` and would shadow these files (including the image tag).

**What stays in the overlays** (not Helm values, so they can't move to `values/`):
the Application object fields — `metadata.name`, the Image Updater `annotations`,
`destination.namespace`, `helm.releaseName`, which `valueFiles` to load — and the
`Ingress` patches.

## ArgoCD Image Updater (dev)

`overlays/dev/kustomization.yaml` adds these annotations:

| Annotation | Value | Meaning |
|---|---|---|
| `image-list` | `api=ghcr.io/.../openeo-argoworkflows-api:dev` | track the `:dev` tag |
| `api.update-strategy` | `digest` | follow that one tag's digest |
| `api.helm.image-name` / `image-tag` | `image.repository` / `image.tag` | Helm keys to write |
| `write-back-method` | `git:secret:argocd/polaris-apps-image-updater` | commit via the SSH deploy key |
| `git-repository` | `git@github.com:...polaris-apps.git` | SSH URL it pushes to |
| `git-branch` | `dev` | branch to commit on |
| `write-back-target` | `helmvalues:/apps/openeo-argoworkflows/values/image-dev.yaml` | file it edits |

### Mutable tags work — no versioning required (yet)

The `digest` strategy tracks the **mutable** `:dev` tag and fires whenever a new
image is pushed to it. It writes `image.tag: dev@sha256:<digest>`, which the chart
renders as `repository:dev@sha256:<digest>` — a valid, digest-pinned reference.
Each push to `:dev` → new digest → a commit to `image-dev.yaml` on `dev` → a
re-sync. To **revert**, `git revert` the `argocd-image-updater` commit (or edit
`image.tag` back and push).

When you later adopt **immutable** tags, switch the strategy in the dev overlay:
- **semver** (`v1.4.2`): `update-strategy: semver` + `api.allow-tags: regexp:^v?[0-9]+\.[0-9]+\.[0-9]+$`
- **date/calver** (`2026-06-29`): `update-strategy: alphabetical`
- **commit-SHA**: `update-strategy: newest-build` (+ an `allow-tags` regex for your SHA format)

### Credentials

Git write-back needs an SSH deploy key with **write** access to polaris-apps,
stored as secret `polaris-apps-image-updater` (key `sshPrivateKey`) in the
`argocd` namespace. Created by ansible from `multinode/vars/secrets.yaml`
(`image_updater_git_ssh_key`); see `multinode/vars/secrets.example.yaml` for how
to generate the key and register the deploy key. github.com host keys come from
the `argocd-ssh-known-hosts-cm` (created by the argo-cd chart, auto-mounted by the
Image Updater chart).

## Not yet tracked

The **executor** image (`global.env.executorImage`) is still pinned manually in
the per-env value files. It can be added to Image Updater later as a second image
in `image-list`, but its `repo:tag` string lives in an env var (not a clean
`image.tag` value), so it needs its own handling.
