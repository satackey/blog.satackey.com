---
path: 2020-03-03-github-actions-docker-buildx-cache
date: 2020-03-03T05:50:34.138Z
title: GitHub ActionsでDocker buildxのマルチステージビルドのキャッシュを保存する方法
description: CIでDockerイメージのビルドをするとき、毎回のビルドに1時間近くかかっていると怠くなってきます(Cross platformなイメージを作ると特に)。
---
CIでDockerイメージのビルドをするとき、毎回のビルドに1時間近くかかっていると怠くなってきます(Cross platformなイメージを作ると特に)。というわけでメモ。

キャッシュの復元/保存の部分を変えればCircleCIなどのほかでも使えると思います。

## コード
```yaml
- name: Restore Docker build cache
  uses: actions/cache@v1
  with:
    path: buildx-cache
    # どのファイルのハッシュにすればいいのかわからないので
    # コミットハッシュを使っています
    key: docker-buildx-${{ github.sha }}
    restore-keys: |
      # docker-buildx-から始まるキャッシュがあれば復元する
      docker-buildx-

- name: Build and push Docker image
  run: |
    set -eux

    # キャッシュが復元されていれば buildx-cache/index.json が存在する。
    # あれば、キャッシュを読み込む引数 を$DOCKER_ARG_CACHE_FROM に代入しておく。
    # 無ければ、空のまま。
    DOCKER_ARG_CACHE_FROM=""
    if [ -e buildx-cache/index.json ];then
      DOCKER_ARG_CACHE_FROM="--cache-from type=local,src=buildx-cache"
    fi
    # キャッシュが存在しないのに、読み込む引数を与えるとエラーで止まってしまう。

    docker buildx build \
      --platform linux/amd64,linux/arm/v7,linux/arm64 \
      --output "type=image,push=true" \
      --file Dockerfile \
      $DOCKER_ARG_CACHE_FROM \ # キャッシュがあれば引数が与えられる
      --cache-to type=local,mode=max,dest=buildx-cache \
      -t your/repo:tag \ #      ↑  # ここがポイント。超重要。
      . # デフォルトはmode=minで、最後のレイヤーしかキャッシュしてくれない。
      # mode=maxを指定すると全てのレイヤーをキャッシュに出力してくれる。
```

## 参考
[Q: --cache-to for multi-stage build · Issue #73 · docker/buildx - GitHub](https://github.com/docker/buildx/issues/73)

[docker/buildx: Docker CLI plugin for extended build capabilities with BuildKit - GitHub](https://github.com/docker/buildx#--cache-tonametypetypekeyvalue)
