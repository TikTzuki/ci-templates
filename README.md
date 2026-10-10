# ci-templates

The deploy pipeline every app shares: **build → tag bump in gitops → Argo CD pulls → verify**.
It's modelled on ewallet's `platform/sre/gitlab-ci-templates`. An app keeps a ~20-line caller
that says *what* (its image, its values file); *how* lives here, versioned by tag.

```
app repo, push to main
  └─ test        the app's own ci.yml (stays in the app: every stack tests differently)
  └─ deploy.yml@v1 (this repo)
       build     tiktuzki/<app>:<sha8> + :latest → Docker Hub         (linux/amd64)
       deploy    <tag_path> = "<sha8>" in <manifest_file> of tiktuzki-gitops, commit, push
       verify    poll <verify_url> until its body contains <sha8>      (optional)
                         │
node1: Argo CD ◄─────────┘ polls gitops, rolls out
```

CI never reaches the cluster: no kubeconfig in CI, and the API server stays private.
**Rollback** = revert the tag commit in the gitops repo.

## Add an app

1. **Gitops:** make sure the app's chart and values file exist in `tiktuzki-gitops`, with an
   image tag at `.image.tag` (or pass `tag_path`), and an Argo CD Application in `apps/dev/`.
2. **Caller:** copy [`examples/deploy.yml`](examples/deploy.yml) to
   `<app>/.github/workflows/deploy.yml` and change the three marked lines.
3. **Secrets:** run
   `pass-cli run --env-file ~/.config/athena/env.tpl -- bin/onboard TikTzuki/<app>`.
4. **Build arg (optional):** to use `verify_url`, have the app report `GIT_SHA`, which every
   build passes as a build arg. In the Dockerfile, put it last so the layer cache survives:
   `ARG GIT_SHA="" / ENV GIT_SHA=$GIT_SHA`, then serve it at a URL such as `/api/version`.

### Inputs

| input | default | |
|---|---|---|
| `image` | — | Docker Hub repo without the tag |
| `manifest_file` | — | values file in the gitops repo |
| `tag_path` | `.image.tag` | yq path of the tag in that file |
| `gitops_repo` / `gitops_branch` | `TikTzuki/tiktuzki-gitops` / `main` | |
| `context` / `dockerfile` | `.` / `<context>/Dockerfile` | e.g. a monorepo: `context: ./checkin` |
| `platforms` | `linux/amd64` | node1 is amd64 |
| `build_args` | — | extra `KEY=value` lines; `GIT_SHA=<sha8>` is always passed |
| `verify_url` / `verify_timeout_minutes` | — / `15` | empty URL = no verify job |

## Secrets: one gitops key for every app

| secret | what | in pass |
|---|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub user | `pass://athena/dockerhub/username` |
| `DOCKERHUB_TOKEN` | Docker Hub access token, Read & Write | `pass://athena/dockerhub/token` |
| `GITOPS_DEPLOY_KEY` | private half of the **one** write deploy key on `tiktuzki-gitops` | `pass://athena/gitops-deploy-key/private_key` |

Personal accounts have no organization-level secrets, so each app repo holds a copy;
`bin/onboard` sets all three in one command. Add these lines to
`~/.config/athena/env.tpl` (the item names above are suggestions; use whatever is in pass):

```
DOCKERHUB_USERNAME=pass://athena/dockerhub/username
DOCKERHUB_TOKEN=pass://athena/dockerhub/token
GITOPS_DEPLOY_KEY=pass://athena/gitops-deploy-key/private_key
```

Create the key once:

```bash
ssh-keygen -t ed25519 -N "" -C ci-templates-gitops -f /tmp/gitops-deploy
# /tmp/gitops-deploy.pub → tiktuzki-gitops ▸ Settings ▸ Deploy keys ▸ Allow write access
# /tmp/gitops-deploy     → Proton Pass item gitops-deploy-key, field private_key
rm /tmp/gitops-deploy /tmp/gitops-deploy.pub
```

**What sharing one key means:** any app repo that holds it can write to *all* of gitops,
and so can deploy anything to node1. To keep that contained:

- only the `deploy` job receives the key;
- that job runs no app code, just two checkouts, `yq`, `sed` and `git push`;
- it runs on pushes to `main` only, never on PRs, so a fork can't reach it.

**Rotating the key:** generate a new pair, replace the deploy key, update pass, then rerun
`bin/onboard` with every app repo.

GitHub accepts a public key as a deploy key on **one** repository only. If you ever have
more than one gitops repo, switch to a fine-grained token or a GitHub App scoped to those
repos.

## Safety rails in deploy.yml

- **Stale runs don't overwrite.** If the deployed tag's commit already contains this run's
  commit, the bump is skipped.
- **One line changes.** The tag line is edited in place. yq finds the line and checks the
  result, and the run fails if more than one line changed. Comments and formatting are kept.
- **Pushes don't collide.** Writers to the same values file are queued, and a push that
  races another app's push is rebased and retried 5 times.
- **Actions are pinned.** Third-party actions are pinned to commit SHAs (they handle the
  Docker Hub token), and Dependabot bumps them monthly.

## Releasing a template change

Callers use `@v1`. Merge to `main`, then move the tag:

```bash
git tag -f v1 && git push -f origin v1
```

A breaking input change gets `v2`, and callers move to it when they are ready.
