---
description: Planner→Generator→Evaluatorのスプリントループを自動実行し、短いプロンプトから製品を組み上げる
argument-hint: "[初回のみ: 1〜4行の製品プロンプト]"
disable-model-invocation: true
model: claude-opus-5-5
---

あなたはPlanner/Generator/Evaluator自律開発パイプラインのオーケストレーターです。以下の手順を順番に実行してください。

## Step 1: 仕様書の確認・作成

1. `docs/spec.md` が存在するか確認する。
2. 存在しない場合:
   - `$ARGUMENTS` が空なら、ユーザーに1〜4行の製品プロンプトを尋ねて停止する。
   - `$ARGUMENTS` があれば、Agentツールで `subagent_type: "agent-pipeline:planner"` を起動し、そのプロンプトを渡して `docs/spec.md` を作成させる。
3. 存在する場合: `$ARGUMENTS` は無視し、既存の `docs/spec.md` からスプリントループを再開する。

## Step 2: 対象スプリントの決定

1. `docs/spec.md` のスプリント一覧を読む。
2. `docs/sprints/sprint-NN.md` を番号順に確認し、YAML frontmatterの `status` が `pass` でない最初のスプリントを対象とする。該当ファイルが無いスプリントは `status: in_progress`, `attempt: 0` とみなす。
3. 全スプリントが `pass` なら、完了報告をしてループを終了する。

## Step 3: Generator → Evaluator の実行

対象スプリントNについて:

1. `attempt` を確認する。`attempt >= 3` かつ直前の判定が `fail` の場合はStep 4（停止）に進む。
2. Agentツールで `subagent_type: "agent-pipeline:generator"` を起動する。対象スプリント番号を渡す。
3. Agentツールで `subagent_type: "agent-pipeline:evaluator"` を起動する。対象スプリント番号を渡す。
4. evaluatorの最終報告（PASS/FAIL判定と基準ごとの詳細）を、あなた自身が `docs/sprints/sprint-NN.md` の末尾に「評価結果」セクションとして追記する（evaluatorはファイルを書かないため、あなたが転記する）。
5. `docs/sprints/sprint-NN.md` のfrontmatterを更新する:
   - 総合判定がPASSなら `status: pass`
   - FAILなら `status: fail`
6. `docs/sprints/sprint-NN.md` を明示的に指定してコミットする（`git add -A` は使わない。リポジトリでなければ省略してよい）。コミットメッセージは `Sprint N attempt M: evaluator verdict <PASS|FAIL>` の形にする。
7. 判定に応じて分岐する:
   - PASS → Step 2に戻り、次のスプリントを処理する。
   - FAIL かつ `attempt < 3` → Step 3の2に戻り、同じスプリントをgeneratorに差し戻す。
   - FAIL かつ `attempt == 3` → Step 4へ。

## Step 4: 停止してユーザーに報告

3回連続で不合格になったスプリントについて、以下をユーザーに報告し、続行方針を確認する:
- スプリント番号と目的
- 3回分のevaluatorの指摘内容の要約
- 提案する選択肢（同じ基準でもう一度試す / このスプリントをスキップして次に進める / 仕様を見直す）

ユーザーの指示を待ち、指示があればそれに従って処理を続ける。

## 全体を通して守ること

- 各スプリントのGenerator・Evaluatorは1回の試行につき1回のAgent呼び出しとする（不要な多重起動をしない）。
- `docs/sprints/sprint-NN.md` の「評価結果」セクションとfrontmatterの`status`以外は書き換えない（実装内容・自己評価はgeneratorが書いたものをそのまま残す）。
- 進捗はこのコマンドを再度実行すれば、`docs/sprints/*.md` のfrontmatterから自動的に再開される。
- Playwright MCPが `⏸ Pending approval` のままだとevaluatorが動かない。`claude mcp list` で `playwright` が `✔ Connected` になっているか確認するよう、必要ならユーザーに伝える。
