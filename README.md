# shared-workflows

複数リポジトリで共有する GitHub Actions 再利用可能ワークフロー（`workflow_call`）集。

各リポジトリにワークフロー全文をコピーする代わりに、ここに集約したワークフローを数行で呼び出す。

## ワークフロー一覧

### dep-check.yml — 依存性チェック

依存性をチェックし、結果を呼び出し元リポジトリの GitHub Issue として通知する。

npm / pnpm / yarn / bun に対応し、lockfile の存在からパッケージマネージャを自動検出する（優先順位 pnpm → yarn → bun → npm）。lockfile が 1 つも無ければ何もせず正常終了する。

- Node.js の EOL チェック（endoflife.date API）
- 検出したパッケージマネージャの最新版チェック（`npm view <pm> version` との比較。yarn berry は npm レジストリの `yarn` が classic 系列を返すため最新版チェックを行わない）
- `npm-check-updates` による patch / minor / major アップデート検出（`--cooldown 7d`: 公開から 7 日未満のバージョンを除外）
- 検出したパッケージマネージャの audit による脆弱性監査（npm `npm audit` / pnpm `pnpm audit --prod` / bun `bun audit` / yarn classic `yarn audit` / yarn berry `yarn npm audit --all --recursive`）
- 変化があれば Issue を作成（同一内容ならスキップ、古いレポート Issue はクローズ）

install は依存の postinstall を実行しない（`--ignore-scripts`、yarn berry は `YARN_ENABLE_SCRIPTS=false`）。cooldown と合わせたサプライチェーン対策。`--target latest` は cooldown 内の最新版を避け、cooldown を満たす最大版を提案する。

Issue は `context.repo` が呼び出し元に解決されるため、呼んだ側のリポジトリに作成される。

#### 呼び出し方

呼び出し側リポジトリの `.github/workflows/dep-check.yml`:

```yaml
name: Dependency Check
on:
  schedule:
    - cron: "0 0 * * 1"
  workflow_dispatch:
permissions:
  contents: read
  issues: write
jobs:
  dep-check:
    uses: yuske-nakajima/shared-workflows/.github/workflows/dep-check.yml@v1
```

トリガー（`schedule` / `workflow_dispatch`）は呼び出し側で定義する。再利用ワークフロー側はトリガーを持たない。

このワークフローは secrets を参照しないため `secrets: inherit` は不要。caller の全シークレットを渡す露出を避けるため指定しない。

#### 前提条件（呼び出し側リポジトリ）

1. lockfile のいずれかがコミットされている（frozen install のため）。優先順位 pnpm → yarn → bun → npm で先頭一致した 1 つを採用する
   - pnpm: `pnpm-lock.yaml`
   - yarn: `yarn.lock`
   - bun: `bun.lockb` または `bun.lock`
   - npm: `package-lock.json`
   - 1 つも無ければワークフローは何もせず正常終了する
2. `.mise.toml` に `node` と使用するパッケージマネージャ（`pnpm` / `yarn` / `bun` 等）の定義がある（`jdx/mise-action@v2` が解決）。npm は Node.js 同梱のため PM 定義は不要
3. caller ワークフローに `permissions: issues: write` を設定する（secrets は参照しないため `secrets: inherit` は不要）
4. `dependencies` ラベルを Issue 用に使用（色分けしたいなら事前にラベル作成）

yarn は `yarn -v` の major で classic(v1) / berry(v2+) を判別し、install・audit のコマンドを切り替える。

`permissions` は再利用ワークフロー側でも宣言しているが、実効権限は caller の `GITHUB_TOKEN` 権限で上限が決まる。caller 側にも `issues: write` が必要。

## バージョン運用

`@v1` で参照する。マージ後に `v1` タグを打つ。

- 新しい v1.x へ追従させたい場合は `git tag -f v1 && git push -f origin v1` で `v1` タグを移動する
- SHA に固定したい場合は caller 側で `@<sha>` 参照に切り替える
- `@main` 参照も動くが、main の変更が全 caller に即時影響するため非推奨
