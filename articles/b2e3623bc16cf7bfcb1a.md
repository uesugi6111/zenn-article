---
title: "DockerFileに最新のNode.jsを追加した話"
emoji: "📘"
type: "tech" 
topics: ["Docker" ,"dockerfile" ,"DockerHub" ,"Node" ,"wasm"]
published: true
---

他で作成していたイメージにNode.jsをインストールしたかったが、調べてもすぐには出てこなかったためメモ

## 前提

Rustでwasmを触ってみようとして必要だった。
busterベースのコンテナを使用。

## 結論

こちらを追加した

```dockerfile
ENV NODE_VERSION 15.0.1

RUN ARCH= && dpkgArch="$(dpkg --print-architecture)" \
    && case "${dpkgArch##*-}" in \
    amd64) ARCH='x64';; \
    ppc64el) ARCH='ppc64le';; \
    s390x) ARCH='s390x';; \
    arm64) ARCH='arm64';; \
    armhf) ARCH='armv7l';; \
    i386) ARCH='x86';; \
    *) echo "unsupported architecture"; exit 1 ;; \
    esac \
    && set -ex \
    && curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-$ARCH.tar.xz" \
    && tar -xJf "node-v$NODE_VERSION-linux-$ARCH.tar.xz" -C /usr/local --strip-components=1 --no-same-owner \
    && rm "node-v$NODE_VERSION-linux-$ARCH.tar.xz" \
    && ln -s /usr/local/bin/node /usr/local/bin/nodejs \
    && node --version \
    && npm --version

```

やっていることとしては
アーキテクチャを取得
公式サイトの[url](https://nodejs.org/ja/download/ "node.sj公式のダウンロード")を叩く
展開

ログに表示されたバージョン

```bash
+ node --version
v15.0.1
+ npm --version
7.0.3
```

## 考えたこと

最初はaptでインストールを行ったがバージョンが古く(10前後だった記憶)、せっかくなので最新にしようと考えた。しかし適当に調べても"正しい"方法が見つからないためDockerHubへ。
https://hub.docker.com/_/node

ここから自分の使っているイメージと似たようなOS(今回はbuster)をベースにしているイメージのDockerFileへ飛ぶ。
そこからnodeをインストールしている箇所を抜き出し、自分のDockerFileに適用。(本当はチェックサムやら何やらがあったが削除してしまった)

見事動いた。
以上。

# 出典

https://hub.docker.com/_/node
https://github.com/nodejs/docker-node/blob/d58d7e65c4f92ef22a190b0ca835ce62464ff3ba/15/buster/Dockerfile
