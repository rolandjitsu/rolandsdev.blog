---
title: "How to enable experimental features for Docker in Github's workflow for Ubuntu 18.04?"
description: "Add buildx and experimental features for Docker in your Github workflows."
date: 2020-10-04T16:36:00+08:00
lastmod: 2022-03-20T19:24:01+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["docker", "github", "github-actions", "ci", "ubuntu"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: no
---

If you're using [Github's workflows](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/introduction-to-github-actions) for CI/CD and you need to use some of Docker's [experimental features](https://github.com/docker/cli/blob/master/experimental/README.md#current-experimental-features), or you want to use [buildx](https://github.com/docker/buildx) or maybe you just want to use some of the new dockerfile [experimental syntaxes](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/experimental.md) then you need to enable the experimental features for the CLI and probably the daemon too.

When running natively on Linux or macOS, it's pretty easy.

To enable the experimental features for the CLI, you just need to add the following to your `~/.docker/config.json` config:
```json
{
  "experimental": "enabled"
}
```

And to enable the experimental features for the daemon, you need to add the following to your `/etc/docker/daemon.json` config:
```json
{
  "experimental": true
}
```

The Github workflow setup is no different:
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

That's pretty much it. See it in action at [rolandjitsu/docker-ssh](https://github.com/rolandjitsu/docker-ssh/blob/master/.github/workflows/test.yml).
