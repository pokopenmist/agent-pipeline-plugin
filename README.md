# agent-pipeline

Planner / Generator / Evaluator による自律開発パイプライン。1〜4行の短いプロンプトから、仕様書生成 → スプリント実装 → 実機テスト（Playwright MCP）のループを自動で回す Claude Code プラグインです。

## 構成

- `agents/planner.md` — 短いプロンプトを `docs/spec.md`（機能一覧＋スプリント計画＋測定可能な受け入れ基準）に展開する
- `agents/generator.md` — 1スプリント分だけを実装し、自己評価を記録する
- `agents/evaluator.md` — Playwright MCPで実機検証し、合否を判定する（ファイルは一切書き換えない）
- `agents/advisor.md` — Codex CLI経由でOpenAIのGPT-6.2 Solに相談する読み取り専用の相談役（合否は決めない）
- `commands/build.md` — 上記エージェントを順番に呼び出すオーケストレーター（`/agent-pipeline:build`）
- `.mcp.json` — Playwright MCPサーバーの登録

### モデルの割り当て（バージョン固定）

| 役 | ファイル | モデル |
|---|---|---|
| オーケストレーター | `commands/build.md` | `claude-opus-5-5` |
| planner | `agents/planner.md` | `claude-opus-5-5` |
| generator | `agents/generator.md` | `claude-sonnet-5-5` |
| evaluator | `agents/evaluator.md` | `claude-sonnet-5-5` |
| advisor（窓口） | `agents/advisor.md` | `claude-sonnet-5-5`（中身の相談先は `gpt-6.2-sol`） |

考える役・判断する役に Opus、手を動かす役に Sonnet を割り当てている。モデルを変える場合は各ファイルの frontmatter の `model:` を書き換える。

### advisor（相談役）

別系統のモデルの目を入れるため、Codex CLI（`codex exec -m gpt-6.2-sol -s read-only`）を呼ぶ。相談するのは次の3か所だけ:

1. 仕様書ができた直後の点検（`spec-review`）。結果は `docs/advice/spec-review.md` に保存し、[重大] の指摘があれば停止してユーザーに確認する
2. FAILで差し戻す前の修正方針の助言（`fix-advice`）。`docs/sprints/sprint-NN.md` に「修正方針の助言」として追記し、generatorが参考にする
3. 3回連続FAILで止まるときの原因の切り分け（`root-cause`）。停止時の報告に含める

advisorの返答は助言であり、合否はevaluatorの判定だけで決まる。Codex CLIが無い・失敗した場合は `ADVISOR_UNAVAILABLE` を返し、パイプラインは advisor なしで従来どおり進む。

> **プラグイン経由のMCPツール名について**: プラグインが`.mcp.json`でバンドルしたMCPサーバーのツール名は、プロジェクト直接登録の場合（`mcp__playwright__*`）とは異なり、`mcp__plugin_<プラグイン名>_<サーバー名>__*`という名前空間になる（実測で確認済み。公式ドキュメントには明記されていない挙動）。`agents/evaluator.md`の`tools:`はこれに合わせて`mcp__plugin_agent-pipeline_playwright__*`としている。プラグイン名を変更する場合はここも合わせて変更すること。

## インストール


### このマシンで試す（ローカルディレクトリから）

`~/.claude/settings.json` に以下を追加する:

```json
{
  "extraKnownMarketplaces": {
    "agent-pipeline-marketplace": {
      "source": {
        "source": "directory",
        "path": "C:\\dev\\agent-pipeline-plugin"
      }
    }
  },
  "enabledPlugins": {
    "agent-pipeline@agent-pipeline-marketplace": true
  }
}
```

またはセッション内で:

```
/plugin marketplace add C:\dev\agent-pipeline-plugin
/plugin install agent-pipeline@agent-pipeline-marketplace
```

開発中に内容を編集した場合、`extraKnownMarketplaces`/`enabledPlugins` 経由だとキャッシュが更新されないことがある。素早く反復するときは `claude --plugin-dir C:\dev\agent-pipeline-plugin` で直接読み込むのが確実。

### GitHubに公開したあと

```
/plugin marketplace add <owner>/<repo>
/plugin install agent-pipeline@agent-pipeline-marketplace
```

## 使い方


新規プロジェクトのディレクトリで:

```
/agent-pipeline:build "2Dレトロゲームメーカーを作って"
```

- 初回は `docs/spec.md` が無ければ `planner` が仕様書を作る。
- 以降はスプリントごとに `generator` → `evaluator` を自動で往復し、`docs/sprints/sprint-NN.md` に記録しながら進む。
- 同一スプリントは最大3回まで試行し、3回連続FAILで停止してユーザーに判断を仰ぐ。
- `/agent-pipeline:build` を再実行すると、`docs/sprints/*.md` のfrontmatterから自動的に再開する。

## 前提条件

- **Codex CLI（advisorを使う場合）**: `codex` コマンドがインストール・ログイン済みで、`gpt-6.2-sol` が使えること。Claudeとは別にOpenAI側の利用料がかかる。1回の相談に数十秒〜数分かかる。未導入でもパイプライン自体は動く。

- **Playwright MCPの承認**: 対話セッションで初めて`claude`を起動したとき、プロジェクトスコープのMCPサーバーを信頼するか確認するプロンプトが一度表示される。`claude mcp list` で `playwright` が `✔ Connected` になっていることを確認してから `/agent-pipeline:build` を実行すること。`⏸ Pending approval` のままだとevaluatorが動かない。
- **`bypassPermissions`は避ける**: 無人実行のために `--permission-mode bypassPermissions` / `--dangerously-skip-permissions` を使うと、後述の権限デナイルールがすべてスキップされる。無人実行が必要な場合は `acceptEdits` など、denyルールを尊重するモードを使うこと。

## 推奨: プロジェクトごとのセキュリティ設定

このプラグイン自体はファイルシステムの安全境界を同梱していない（プラグインの `settings.json` は `agent`/`subagentStatusLine` のみ対応で、`permissions` は同梱できない仕様のため）。`/agent-pipeline:build` を使うプロジェクトの `.claude/settings.json` に、以下のような設定を個別に追加することを推奨する:

```json
{
  "permissions": {
    "deny": [
      "Edit(~/.ssh/**)",
      "Edit(~/.aws/**)",
      "Edit(~/.azure/**)",
      "Edit(~/Documents/**)",
      "Edit(~/Desktop/**)",
      "Edit(~/.claude/settings.json)",
      "Edit(.claude/settings.json)",
      "Edit(.claude/settings.local.json)"
    ],
    "disableBypassPermissionsMode": "disable"
  }
}
```

**既知の限界**: 上記はClaude自身のWrite/Edit/NotebookEditツール呼び出しと、Bash内の一部の認識済みコマンド（`cat`, `head`, `tail`, `sed`）にしか効かない。`node`/`npm`/`npx`/Pythonスクリプトなど、generatorやevaluatorが実際に多用するBash経由のサブプロセスがファイルを直接書き込む操作は防げない。これをOSレベルで強制する`sandbox`機能はmacOS/Linux/WSL2のみ対応で、ネイティブWindowsでは使えない（2026-08-03時点）。真に強い保証が必要な場合はWSL2上でClaude Codeを動かすこと。

## 状態管理

- 別途のJSON状態ファイルは持たない。`docs/spec.md` と `docs/sprints/sprint-NN.md` のYAML frontmatter（`status`, `attempt`）だけで進捗を追跡する。
- パイプライン実行中は `docs/spec.md` / `docs/sprints/sprint-NN.md` を人間が直接編集しない（attempt/statusの整合性が壊れる）。

## Gitコミットの規律

各エージェント定義に、いつコミットするかを埋め込んでいる:

- **planner**: `docs/spec.md` を書き終えたらコミット（`Add product spec: <製品名>`）
- **generator**: 実装＋自己評価を書き終えたらコミット（`Sprint N attempt M: <実装内容の要約>`）
- **build（オーケストレーター）**: advisorの仕様書点検を保存したらコミット（`Add advisor spec review`）。evaluatorの評価結果（とFAIL時のadvisorの助言）を転記しstatusを更新したらコミット（`Sprint N attempt M: evaluator verdict <PASS|FAIL>`）

各コミットは対象ファイルを明示的に `git add` してから行う（`git add -A` は使わない）。


