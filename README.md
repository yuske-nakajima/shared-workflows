# shared-workflows

複数リポジトリで共有する GitHub Actions 再利用可能ワークフロー（`workflow_call`）集。

各リポジトリにワークフロー全文をコピーする代わりに、ここに集約したワークフローを数行で呼び出す。

## ワークフロー一覧

### dep-check.yml — 依存性チェック

依存性をチェックし、結果を呼び出し元リポジトリの GitHub Issue として通知する。

- Node.js の EOL チェック（endoflife.date API）
- pnpm バージョン整合性チェック（`.mise.toml` / `packageManager` / `engines.pnpm` の一致）
- `npm-check-updates` による patch / minor / major アップデート検出（`--cooldown 7d`: 公開から 7 日未満のバージョンを除外）
- `pnpm audit` による脆弱性監査
- 変化があれば Issue を作成（同一内容ならスキップ、古いレポート Issue はクローズ）

cooldown はサプライチェーン対策。`--target latest` は cooldown 内の最新版を避け、cooldown を満たす最大版を提案する。caller 側の pnpm 設定（`minimumReleaseAge`）は不要。

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

1. `.mise.toml` に `node` と `"npm:pnpm"` のバージョン定義がある（`jdx/mise-action@v2` が解決）
2. pnpm プロジェクトで、`package.json` に `packageManager`（`pnpm@x.y.z`）と `engines.pnpm` がある
3. lockfile（`pnpm-lock.yaml`）がコミットされている（`--frozen-lockfile` のため）
4. caller ワークフローに `permissions: issues: write` を設定する（secrets は参照しないため `secrets: inherit` は不要）
5. `dependencies` ラベルを Issue 用に使用（色分けしたいなら事前にラベル作成）

`permissions` は再利用ワークフロー側でも宣言しているが、実効権限は caller の `GITHUB_TOKEN` 権限で上限が決まる。caller 側にも `issues: write` が必要。

## バージョン運用

`@v1` で参照する。マージ後に `v1` タグを打つ。

- 新しい v1.x へ追従させたい場合は `git tag -f v1 && git push -f origin v1` で `v1` タグを移動する
- SHA に固定したい場合は caller 側で `@<sha>` 参照に切り替える
- `@main` 参照も動くが、main の変更が全 caller に即時影響するため非推奨
