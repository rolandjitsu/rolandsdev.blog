---
title: "How to access private Github repos when building Docker images?"
description: "Access private repositories from your Dockerfile without leaving behind any ssh keys or credentials."
date: 2020-10-03T20:06:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["docker", "github", "ci"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: true
---

If you need to access private GitHub repositories while building a Docker image without leaving SSH keys or tokens embedded in intermediate layers or the final image, [BuildKit](https://github.com/moby/buildkit)'s SSH mount feature (`RUN --mount=type=ssh`) provides a secure, leak-free solution.

In modern Docker (v23+), BuildKit is the default build engine and SSH mounts are fully stable; you no longer need to enable experimental flags for the CLI or the daemon.

## Setup SSH Agent

Ensure your SSH agent is running and has your private key loaded:

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa # or id_ed25519
```

Create a `known_hosts` file with GitHub's current host keys:

```bash
ssh-keyscan -t ed25519,ecdsa,rsa github.com > ./known_hosts
```

## Dockerfile with SSH Mount

Use the official Dockerfile syntax pragma `# syntax=docker/dockerfile:1`:

```dockerfile
# syntax=docker/dockerfile:1
# Dockerfile.priv-repo
FROM alpine:3.20

ARG PRIV_GIT_REPO
ENV PRIV_SOURCE_CODE=/code

RUN apk add --no-cache ca-certificates git openssh-client rsync

# Mount the host SSH agent socket and known_hosts file during build execution only
RUN --mount=type=ssh \
    --mount=type=secret,id=known_hosts,dst=/root/known_hosts \
    mkdir -p -m 0700 /root/.ssh && \
    cat /root/known_hosts >> /root/.ssh/known_hosts && \
    git config --global url."git@github.com:".insteadOf "https://github.com/" && \
    git clone "${PRIV_GIT_REPO}" "${PRIV_SOURCE_CODE}"
```

Build the intermediate image forwarding your SSH agent:

```bash
docker buildx build -f Dockerfile.priv-repo \
    --ssh default \
    --secret id=known_hosts,src=./known_hosts \
    --build-arg PRIV_GIT_REPO="git@github.com:my-org/my-private-repo.git" \
    --tag priv-repo \
    .
```

{{< admonition type=note >}}
The `--ssh default` flag forwards the local `SSH_AUTH_SOCK` into the build container for the duration of the `RUN` command only. The socket is not accessible during subsequent build steps or stored in image metadata.
{{< /admonition >}}

## Exporting files to host (Optional)

If your goal is extracting repository artifacts to your local disk without retaining container layers:

```dockerfile
# Dockerfile
FROM priv-repo AS builder

RUN rsync -av --exclude '.git' $PRIV_SOURCE_CODE/ /tmp/out/

FROM scratch
COPY --from=builder /tmp/out/ /
```

Run Buildx with a local exporter:

```bash
docker buildx build -f Dockerfile -o type=local,dest=./priv-code .
```

The cloned source files will be written directly to `./priv-code` on your host.

## Notes & Best Practices

- **Docker Bake (`docker buildx bake`)**: When building with Bake, specify `ssh = ["default"]` on the target and pass `--allow=ssh` on the CLI to grant Buildx the privilege to forward the host SSH socket. If passing explicit socket paths outside your workspace or hitting filesystem entitlement restrictions in Buildx v0.20+, set `BUILDX_BAKE_ENTITLEMENTS_FS=0` or pass `--allow=fs.read=<path>`.
- **GitLab / Bitbucket**: This pattern works across all Git providers by swapping `github.com` in `known_hosts` and the `insteadOf` configuration.
- The sample code and GitHub Actions workflow are available at [rolandjitsu/docker-ssh](https://github.com/rolandjitsu/docker-ssh).

## Security Posture & Authentication Choices

### Isolating SSH Access in Builds

1. **Least privilege**: Forward the SSH agent socket only to targets that perform clones (such as `priv-repo`). Downstream stages (compiling, packaging) should not declare `ssh` attributes or request socket access.
2. **Ephemeral access**: `--mount=type=ssh` exposes the socket only during that single `RUN` instruction. The socket is not accessible during later build steps and is never persisted in image layers or metadata.
3. **Exclude Git metadata**: When copying cloned repositories into final images or host directories, exclude `.git/` (`rsync --exclude '.git'`). This avoids leaking commit histories, author metadata, or remote URLs into production artifacts.
4. **Enforce host verification**: Mount a trusted `known_hosts` file via `--mount=type=secret` to protect against man-in-the-middle attacks.

### Deploy Keys vs GitHub Apps

When choosing how machine credentials authenticate to private repositories in CI/CD pipelines, there are two primary approaches:

| Feature | Deploy Keys (SSH) | GitHub Apps (Token) |
| --- | --- | --- |
| **Credential Type** | Static SSH key pair | Short-lived installation token (~1 hr) |
| **Scope** | Single repository per key | Multi-repository across an organization |
| **Expiration** | Manual rotation | Automatic |
| **BuildKit Integration** | `--mount=type=ssh` | `--mount=type=secret` |
| **Audit Log Trail** | Key fingerprint | App name + actor |

#### When using Deploy Keys

- Always use modern `ed25519` keys (`ssh-keygen -t ed25519`).
- GitHub deprecated and disabled legacy SHA-1 RSA keys in March 2022. Older keys without SHA-2 signatures fail authentication with `Permission denied (publickey)`.
- Deploy keys are tied to a single repository. If your build needs to clone multiple private dependencies or submodules across an organization, managing unique deploy keys per repository becomes difficult to maintain.

#### Recommended: GitHub Apps

For organization-scale CI/CD, GitHub Apps are the recommended solution:

1. **Short-lived credentials**: Using the official [actions/create-github-app-token](https://github.com/actions/create-github-app-token) action generates an installation access token valid for 60 minutes. There are no static credentials that can be forgotten or leaked indefinitely.
2. **Multi-repo access**: A single GitHub App installation can be granted `contents: read` across multiple private dependency repositories.
3. **BuildKit integration**: Pass the generated token via `--mount=type=secret` and rewrite Git URLs:

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20

RUN apk add --no-cache ca-certificates git

RUN --mount=type=secret,id=github_token \
    TOKEN=$(cat /run/secrets/github_token) && \
    git config --global url."https://x-access-token:${TOKEN}@github.com/".insteadOf "https://github.com/" && \
    git clone https://github.com/my-org/my-private-repo.git /code
```

Invoke buildx passing the token from your workflow environment:

```bash
docker buildx build --secret id=github_token,env=GITHUB_TOKEN .
```
