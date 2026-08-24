# Custom Assembly — base + overlay GitOps example

A working example of reconciling [Chainguard Custom Assembly](https://edu.chainguard.dev/chainguard/chainguard-images/features/custom-assembly/)
configs across many repos from a single GitHub Actions workflow, using a
**shared base config + per-repo overlays**.

`chainctl images repos build apply` is **declarative and a full replace**. This repository demonstrates
how you can use a base config to apply to all (or a grouping of images), and then also apply specific config
to only one (or a selected grouping) of images on top of the base - without over-writing the base config. The
example used here will apply custom CA-certs to all images, and then add labels and environment variables
to repos that are specific to those repositories. 

## Directory

```
ca-config/
  base.yaml              # shared: internal root CA(s) for ALL repos
  overlays/
    python.yaml          # python-only: pip index, annotations
    node.yaml            # node-only: npm registry, annotations
    # (no go.yaml)       # go gets base-only — demonstrates the fallback (no overlay / additional config aside from base applied)
.github/workflows/
  custom-assembly.yml    # merges base + overlay, applies once per repo
```

For each repo the workflow deep-merges `base.yaml` with `overlays/<repo>.yaml`
using [`yq`](https://github.com/mikefarah/yq), then applies the **merged**
result. Every apply is the full desired state, so:

- `python` and `node` get **certs + their own annotations/env** in one apply.
- `go` (no overlay) gets **just the certs**.
- Re-running is **idempotent** — nothing ever gets clobbered by a partial apply.

The merge uses `*+`, which **appends arrays** (e.g. package lists) rather than
replacing them, so a package defined in both `base.yaml` and an overlay won't
overwrite the other.

## PR vs. main behavior

| Trigger | Command | Effect |
| --- | --- | --- |
| Pull request | `apply --dry-run` | Drift detection — prints the diff, **fails if changes are pending**. Nothing is applied. |
| Push to `main` / manual dispatch | `apply --yes` | Applies the merged config and kicks off the build for each repo. |

## Setup

1. **Enable Custom Assembly certificates (Beta)** for your org — contact your
   Chainguard Customer Success team. (Only needed for the `certificates` block.)

2. **Create a Chainguard identity for GitHub OIDC** and grant it `repo.list` /
   `repo.update` on your org:

   ```sh
   chainctl iam identities create github ca-gitops \
     --github-repo="<your-org>/<this-repo>" \
     --github-ref="refs/heads/main" \
     --role=registry.editor \
     --parent=example.com
   ```

   Put the resulting identity ID in a repo secret named `CHAINGUARD_IDENTITY`.

3. **Edit the config for your org:**
   - Set `ORGANIZATION` and `REPOS` in `.github/workflows/custom-assembly.yml`.
   - Replace the example CA in `ca-config/base.yaml` with your real PEM cert.
   - Add/adjust overlays in `ca-config/overlays/`.

## Try the merge locally

```sh
yq eval-all '. as $item ireduce ({}; . *+ $item)' \
  ca-config/base.yaml ca-config/overlays/python.yaml
```

You'll see the CA from `base.yaml` plus python's annotations/env/packages in a
single document — exactly what the workflow applies.
