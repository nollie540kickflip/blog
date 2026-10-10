---
layout: post
title: "【CI/CD】オンプレGitLabでコンテナレジストリを構築してKanikoでビルドする完全ガイド🐳"
date: 2026-10-10 10:00:00 +0900
categories: [CI/CD, GitLab]
tags: [gitlab, registry, kaniko, fastapi, docker, colima]
mermaid: false
---

こんにちは！最近、プライベートのオンプレ環境に構築した GitLab で「コンテナレジストリ（Container Registry）」を有効化し、CI/CD から自動でイメージをビルド＆プッシュする仕組みを作りました！🎉

この構成、実はDockerの特権モード（`privileged`）を使わずに安全にビルドできる **Kaniko** を採用していたり、セマンティックバージョニング（v1.2.3みたいなタグ付け）に自動対応させたりと、なかなか実用的な環境に仕上がっています。

今回は、備忘録も兼ねてその一連の手順と設定をまるっと大公開します！💪

---

## 🏗️ 1. アーキテクチャ概要

今回の構成はこんな感じです：

*   **GitLab サーバー**: Docker Compose 上で稼働（IP: `192.168.0.100`）
    *   Web/API: `http://192.168.0.100:80`
    *   Container Registry: `http://192.168.0.100:5050` (今回は手軽なHTTP通信を使用)
*   **CI/CD ビルドエンジン**: **Kaniko**（特権モード不要、HTTP レジストリプッシュ対応！）
*   **Web アプリケーション**: FastAPI (Python 3.11) をコンテナ化してデプロイ
*   **クライアント**: macOS (Homebrew Docker CLI + Colima)

---

## ⚙️ 2. GitLab 側の設定（Container Registry 有効化）

まずは GitLab 側でコンテナレジストリの機能をオンにします。Docker Compose の設定ファイル（`compose.yaml`）を少し書き換えます。

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: 'gitlab.local'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://192.168.0.100'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222

        # --- 👇ここから Container Registry 設定 👇 ---
        registry_external_url 'http://192.168.0.100:5050'
        registry['enable'] = true
        gitlab_rails['registry_enabled'] = true
        # ----------------------------------------
    ports:
      - '80:80'
      - '443:443'
      - '2222:22'
      - '5050:5050' # 👈 レジストリ用のポートを忘れずに開放！
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
    shm_size: '256m'
```

### 💡 ここでの超重要ポイント！
1. `registry_external_url` の後ろには **`=`（イコール）を付けない** こと！（関数呼び出しの構文のためエラーになります）
2. `registry['enable'] = true` を入れ忘れると、バックグラウンドのレジストリサービス自体が起動してくれません。

設定を追記したら、以下のコマンドで反映させます。

```bash
# コンテナの再作成
$ docker compose up -d

# 設定の再反映
$ docker exec -it gitlab gitlab-ctl reconfigure

# 起動確認 (一覧に run: registry があれば大成功✨)
$ docker exec -it gitlab gitlab-ctl status
```

---

## 🐍 3. テスト用の FastAPI アプリを用意

今回はサンプルとして、軽量で使いやすい FastAPI のアプリを用意しました。

**`main.py`**
```python
from fastapi import FastAPI

app = FastAPI(title="FastAPI GitLab Registry Demo")

@app.get("/")
def read_root():
    return {"message": "Hello from FastAPI hosted on GitLab Container Registry!"}

@app.get("/health")
def health_check():
    return {"status": "ok"}
```

**`Dockerfile`**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 🚀 4. Kaniko で安全にビルドする GitLab CI パイプライン

ここが今回の目玉です！
よくある Docker-in-Docker（dind）は、Runner に特権モード（`privileged = true`）を付与する必要がありセキュリティ的に少し不安ですよね。そこで今回は、特権不要でコンテナをビルドできる **Kaniko** を使います。

さらに、Gitで `v1.2.3` のようなタグを打つと、自動で `1.2.3` `1.2` `1` `latest` などのセマンティックバージョニングのタグを一括生成する賢いスクリプトを組み込みました！

**`.gitlab-ci.yml`**
```yaml
stages:
  - build

build-image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.14.0-debug
    entrypoint: [""]
  before_script:
    # GitLab CI が自動注入する変数で認証情報を作成
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
  script:
    - |
      # セマンティックバージョニングの判定と動的タグ生成
      if [ -n "$CI_COMMIT_TAG" ]; then
        # 先頭の 'v' を除外 (例: v1.2.3 -> 1.2.3)
        SEMVER="${CI_COMMIT_TAG#v}"
        MAJOR=$(echo "$SEMVER" | cut -d. -f1)
        MINOR=$(echo "$SEMVER" | cut -d. -f2)
        PATCH=$(echo "$SEMVER" | cut -d. -f3)

        # SemVer タグの追加
        DESTINATIONS="--destination ${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
        DESTINATIONS="$DESTINATIONS --destination ${CI_REGISTRY_IMAGE}:${SEMVER}"
        if [ -n "$MAJOR" ] && [ -n "$MINOR" ]; then
          DESTINATIONS="$DESTINATIONS --destination ${CI_REGISTRY_IMAGE}:${MAJOR}.${MINOR}"
        fi
        if [ -n "$MAJOR" ]; then
          DESTINATIONS="$DESTINATIONS --destination ${CI_REGISTRY_IMAGE}:${MAJOR}"
        fi
        DESTINATIONS="$DESTINATIONS --destination ${CI_REGISTRY_IMAGE}:latest"
      else
        # 通常のブランチ push 時
        DESTINATIONS="--destination ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA} --destination ${CI_REGISTRY_IMAGE}:latest"
      fi
    # Kaniko によるビルド・プッシュ (--insecure で HTTP レジストリに対応)
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      $DESTINATIONS
      --insecure
  rules:
    - if: $CI_COMMIT_TAG =~ /^v?[0-9]+\.[0-9]+\.[0-9]+/
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_BRANCH == "main"
```

これだけで、`git push origin v1.0.0` のようにリリースタグを打つだけで、美しいバージョニングが付いたコンテナイメージがレジストリに登録されるようになります！😎

---

## 💻 5. クライアント（Mac + Colima）からプルしてみる

最後に、手元の Mac から GitLab のレジストリにアクセスして、作ったイメージを動かしてみます。
今回は HTTP のレジストリ（`http://192.168.0.100:5050`）に繋ぐので、Docker デーモン側で **insecure-registry（安全でないレジストリ）** として許可してあげる必要があります。

Colima を使っている場合は以下の手順で設定します。

```bash
# 1. Colima の設定ファイルを開く
$ colima start --edit

# 2. docker セクションに insecure-registries を追記
docker:
  insecure-registries:
    - 192.168.0.100:5050

# 3. Colima を再起動
$ colima stop
$ colima start
```

準備ができたら、ログインしてプル＆実行です！

```bash
# レジストリにログイン (GitLabのユーザー名・パスワードでOK)
$ docker login 192.168.0.100:5050

# コンテナを起動！
$ docker run -d -p 8000:8000 --name fastapi-app 192.168.0.100:5050/dev/test:latest
```

ブラウザで `http://localhost:8000/docs` にアクセスして、FastAPI の Swagger UI が表示されれば大成功です✨

---

## 🚑 6. ハマりやすいトラブルシューティング

構築中に実際に私がハマったポイントとその解決策も残しておきます。

| 😭 起きた現象 | 🔍 主な原因 | 🛠️ 解決策 |
|---|---|---|
| `docker login` で Docker Hub (`registry-1.docker.io`) に繋がり認証失敗する | GitLab 側でレジストリが無効のため `$CI_REGISTRY` が空になっていた | `compose.yaml` で `registry['enable'] = true` などを正しく設定し `reconfigure` を実行する |
| `gitlab-ctl status` に `registry` が現れない | `registry_external_url = '...'` のように `=` が入っていた、または `enable` が抜けていた | `=` を削除し、`registry['enable'] = true` を追記する |
| CI で `mount: permission denied (are you root?)` が発生する | `docker:dind` に特権モード（`privileged = true`）が必要だった | 特権不要の **Kaniko** にビルドエンジンを切り替える（今回の構成！） |
| プル時に `http: server gave HTTP response to HTTPS client` と怒られる | Docker クライアントが HTTP レジストリへの接続を拒否している | Colima や Docker の `insecure-registries` に `192.168.0.100:5050` を登録する |


オンプレミスでの CI/CD やレジストリ構築は少しハードルが高いですが、一度作ってしまえば完全に自分だけの遊び場になるので最高ですね！
ぜひ参考にしてみてください〜！🙌
