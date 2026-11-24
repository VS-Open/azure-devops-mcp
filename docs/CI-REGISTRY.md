# CI/CD: tests on `dev`, Docker image published to the container registry on tag

Workflow: [`.github/workflows/val-ci-registry.yml`](../.github/workflows/val-ci-registry.yml)

| Event                            | What runs                                                                  |
| -------------------------------- | -------------------------------------------------------------------------- |
| Push / PR on `dev`               | lint, format check, build, unit tests (`npm test`), Docker build (no push) |
| Tag `vX.Y.Z` (on a `dev` commit) | everything above, then build and **push the image to the registry**        |

The self-hosted integration tests (`npm run test:selfhosted`) are never run by the pipeline.

## Release

```bash
git checkout dev && git pull
git tag v2.11.1-val.1      # SemVer; a "-suffix" is treated as a pre-release (no `latest` / `major.minor` tags)
git push origin v2.11.1-val.1
```

Published tags: `2.11.1-val.1` (and, for non pre-release tags, `2.11` and `latest`). A tag whose commit is not on `dev` is refused.

## Configuration (nothing is hard-coded)

Repository **secrets** (Settings → Secrets and variables → Actions). They are secrets rather than variables so they are masked in logs even if the repository is public:

| Secret              | Example / purpose                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------- |
| `REGISTRY_URL`      | Registry host (`registry.example.com`); `REGISTRY_HOST` is accepted as an alias, a scheme is stripped |
| `REGISTRY_PROJECT`  | Registry project / namespace (`my-project`)                                                           |
| `REGISTRY_IMAGE`    | Optional image name (default `azure-devops-mcp`)                                                      |
| `REGISTRY_USERNAME` | Registry robot / service account                                                                      |
| `REGISTRY_PASSWORD` | Robot account secret                                                                                  |

Repository **variables** (optional):

| Variable       | Purpose                                                                                                                          |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `RUNS_ON`      | JSON array of runner labels, e.g. `["self-hosted","linux"]` (default `["ubuntu-latest"]`, normally enough with a cloud registry) |
| `NPM_REGISTRY` | Internal npm registry/proxy used by `npm ci`                                                                                     |

Optional: create an Environment named `registry` (Settings → Environments) to require a manual approval before publishing.

## Prerequisites and cautions

- **Runners:** The registry is a cloud service, so the default GitHub-hosted runners (`ubuntu-latest`) are enough. Set `RUNS_ON` only if you want to use your own runners; in that case never let pull requests from forks run on them.
- **Robot account:** create it in the registry project with only the `push` / `pull` permission on repositories.
- **Public repository:** a fork of a public repository is public, and so are its Actions logs. Registry host and credentials are stored as secrets so they stay masked. The jobs are already restricted to PRs from the same repository; also set _Settings → Actions → General → Fork pull request workflows_ to require approval.
- **Upstream workflows:** disable the inherited Microsoft workflows (`build`, `codeql`, `release`, `pre-release`, `version-update`, `dependency-review`) in the Actions tab rather than deleting them, so merging `upstream/main` stays conflict-free.
- **Base image:** the Dockerfile pulls `node:20-slim` from Docker Hub. If your runners cannot reach it, mirror it in the registry and adapt the `FROM` line.
- **Actions pinning:** the Docker actions are pinned by major version; pin them by commit SHA if your security policy requires it.
