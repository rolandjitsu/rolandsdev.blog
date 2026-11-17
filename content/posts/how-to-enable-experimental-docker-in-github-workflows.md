---
title: "How to enable experimental features for Docker in Github's workflow for Ubuntu 18.04?"
description: "Add buildx and experimental features for Docker in your Github workflows."
date: 2020-10-04T16:36:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["docker", "github", "github-actions", "ci", "ubuntu"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: true
---

{{< admonition type=warning title="Deprecated Setup" >}}
This post is preserved for historical context. In modern Docker (v23+) and GitHub Actions:

- **Buildx and BuildKit are enabled by default**: You no longer need to enable experimental flags for the Docker CLI or daemon to use Buildx, multi-stage builds, or `--mount=type=ssh/secret`.
- **`ubuntu-18.04` is removed**: GitHub-hosted runners retired Ubuntu 18.04 in December 2022. Workflows should use `ubuntu-latest` (or `ubuntu-24.04`).
- **Official GitHub Actions**: The canonical modern workflow uses official actions like [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action) and [docker/setup-qemu-action](https://github.com/docker/setup-qemu-action):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - run: docker buildx build .
```

{{< /admonition >}}

If you're using [Github's workflows](https://docs.github.com/en/actions) for CI/CD and you need to use some of Docker's [experimental features](https://docs.docker.com/), or you want to use [buildx](https://github.com/docker/buildx) or Dockerfile frontend syntaxes on legacy runners, you needed to enable the experimental features for the CLI and daemon.

When running natively on Linux or macOS:

To enable the experimental features for the CLI, you add the following to `~/.docker/config.json`:

```json
{
  "experimental": "enabled"
}
```

And for the daemon, in `/etc/docker/daemon.json`:

```json
{
  "experimental": true
}
```

On legacy GitHub workflows, the step was configured via inline shell commands:

```yaml
name: Stuff builder
on:
  push:
    branches:
      - master

jobs:
  build-stuff:
    runs-on: ubuntu-18.04
    steps:
      - name: Enable experimental features for the Docker daemon and CLI
        run: |
          echo $'{\n  "experimental": true\n}' | sudo tee /etc/docker/daemon.json
          mkdir -p ~/.docker
          echo $'{\n  "experimental": "enabled"\n}' | sudo tee ~/.docker/config.json
          sudo service docker restart
          docker version -f '{{.Client.Experimental}}'
          docker version -f '{{.Server.Experimental}}'
          docker buildx version
```

See the example repository at [rolandjitsu/docker-ssh](https://github.com/rolandjitsu/docker-ssh).
