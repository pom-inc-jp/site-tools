# site-tools

株式会社ポムの店舗サイト（casse.cafe・retopo.cafe・selphoto.studio など）で共通に使う仕組み。今は「書き出しと公開」のワークフローだけを置いている。

## 書き出しと公開（`.github/workflows/deploy-static.yml`）

各店のリポジトリから呼び出す。中身は次のとおりで、どの店でも同じ。

1. `node build.mjs` で `dist/` に書き出す
2. 公開先の設定があれば、残すフォルダを調べ、試し（dry-run）で「消えるファイル」「追加・更新されるファイル」の一覧を実行結果に出す
3. 反映してよいときだけ、rsync で公開フォルダを `dist/` と同じにする

### どの店でも消さないもの

- 名前が `_movie_app` で終わるフォルダ、`movie/`、`movie_app/`（店内iPadの動画など。中身もすべてそのまま）
- 反映の直前にサーバーを調べ、mp4 が入っているフォルダすべて
- `.well-known/`（SSL証明書の更新に使う）

店ごとに残したいものは、呼び出し側の `keep` に書く。

### 呼び出し方（各店の `.github/workflows/deploy.yml`）

```yaml
name: 公開

on:
  push:
    branches: [main]
  schedule:
    - cron: '10 18 * * *'   # 毎日 3:10（日本時間）。お知らせの「NEW」を日付で外すため
  workflow_dispatch:
    inputs:
      mode:
        description: 'dry-run＝一覧を見るだけ／deploy＝反映する'
        type: choice
        options: [dry-run, deploy]
        default: dry-run

concurrency:
  group: deploy
  cancel-in-progress: false

jobs:
  deploy:
    uses: pom-inc-jp/site-tools/.github/workflows/deploy-static.yml@v1
    with:
      live: ${{ vars.CASSE_LIVE == 'true' }}
      mode: ${{ github.event_name == 'workflow_dispatch' && inputs.mode || 'auto' }}
      keep: |
        /casse_app/
        /wp-content/uploads/
    secrets: inherit
```

| 入力 | 意味 |
| --- | --- |
| `live` | 本番に切り替え済みか。`true` のときだけ、手動以外（main への反映・毎日の書き出し）でも反映する |
| `mode` | `auto`（live に従う）／`dry-run`（一覧だけ）／`deploy`（この1回だけ反映）。手動実行のときは選んだもの、それ以外は `auto` を渡す |
| `keep` | 店ごとに残すフォルダ・ファイル。1行に1つ、rsync の除外の書き方（先頭の `/` は公開フォルダ直下の意味） |
| `node-version` | 省略時は 20 |
| `build-command` | 省略時は `node build.mjs` |

### 公開先の接続情報

全店が同じ Xserver アカウントなので、接続情報と鍵は1つを共通で使う。

**組織（pom-inc-jp）の Secrets に1回だけ置くもの**（Organization settings → Secrets and variables → Actions → New organization secret。使えるリポジトリは「Selected repositories」で各店を選ぶ。無料プランでは公開リポジトリだけが組織の Secrets を使えるので、このリポジトリと各店のリポジトリは公開にしている）

- `XSERVER_HOST`（Xserver のホスト名。サーバーパネルの「サーバー情報」にある sv〇〇〇〇.xserver.jp）
- `XSERVER_USER`（サーバーID）
- `XSERVER_SSH_KEY`（自動公開用の秘密鍵。チャットなどには貼らない）

**公開フォルダ（`XSERVER_PATH`）は書かなくてよい。** 書かなければ `/home/<XSERVER_USER>/<リポジトリ名>/public_html/` に公開する（リポジトリ名をドメイン名にしているため）。別の場所に出したい店だけ、その店のリポジトリの環境 production に `XSERVER_PATH` を置く。形が `/home/〇〇/〇〇/public_html/` でないときは、間違って別の場所を消さないように止まる。

店のリポジトリの環境 production に同じ名前の Secrets があると、そちらが優先される。組織にまとめたあとは、店ごとの古い Secrets は消しておく（どちらが使われているか分からなくならないように）。

設定がないうちは、書き出しだけで終わる（失敗にはしない）。

### 使えるようにする設定（一度だけ）

このリポジトリは非公開なので、「Settings → Actions → General → Access」で「Accessible from repositories in the 'pom-inc-jp' organization」を選んでおく。

## 変えるとき

- 呼び出し側は `@v1` で固定している。`v1` は「各店が使ってよい版」を指すブランチ。中身を変えたら、まず main で直し、どこか1店で `@main` を指して試す。問題なければ `v1` ブランチを main と同じにする（プルリクエストで main → v1 をマージ）
- 呼び出し側の書き方が変わるような大きな変更は `v2` として出し、各店を順に切り替える
- 残すフォルダ（どの店でも消さないもの）を減らす変更はしない。店内iPadの動画が消えるおそれがある

## 使っている店

| 店 | リポジトリ | 状態 |
| --- | --- | --- |
| CASSE | pom-inc-jp/casse.cafe | 共通版を使う（2026年11月〜） |
| RETOPO | pom-inc-jp/retopo.cafe | まだ自前のワークフロー。CASSEで問題ないことを確かめてから切り替える |
| Selphoto Studio | pom-inc-jp/selphoto.studio | 同上 |
