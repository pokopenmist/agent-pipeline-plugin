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

## Step 1.5: 仕様書の点検（advisor）

`docs/advice/spec-review.md` が既に存在する場合はこのStepを飛ばしてStep 2へ進む。

1. Agentツールで `subagent_type: "agent-pipeline:advisor"` を起動し、相談の種類 `spec-review` を渡す。
2. 返答が `ADVISOR_UNAVAILABLE` で始まる場合は、その理由をユーザーに一言伝え、点検なしでStep 2へ進む（ファイルは作らない。次回の実行で再度点検を試みる）。
3. それ以外の場合は、返答をそのまま `docs/advice/spec-review.md` に書き、そのファイルだけを明示的に指定してコミットする（メッセージ: `Add advisor spec review`）。
4. 返答の最終行が `VERDICT: BLOCKING` なら停止し、[重大] の指摘をユーザーに示して、次のどちらにするかを尋ねる:
   - 仕様書を見直す（ユーザーが `docs/spec.md` を直してから `/agent-pipeline:build` を再実行する）
   - このまま進める（ユーザーが `/agent-pipeline:build` を再実行すると、点検済みとしてStep 2から進む）
5. `VERDICT: OK`（または判定行が無い）なら、[軽微] の指摘があれば一言だけ報告してStep 2へ進む。

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
6. FAIL かつ `attempt < 3` の場合のみ、Agentツールで `subagent_type: "agent-pipeline:advisor"` を起動し、相談の種類 `fix-advice` と対象スプリント番号を渡す。
   - 返答が `ADVISOR_UNAVAILABLE` で始まらなければ、`docs/sprints/sprint-NN.md` の末尾に「修正方針の助言 (Attempt M, advisor)」セクションとして返答をそのまま追記する。
   - `ADVISOR_UNAVAILABLE` なら何も追記せず、そのまま進む。
7. `docs/sprints/sprint-NN.md` を明示的に指定してコミットする（`git add -A` は使わない。リポジトリでなければ省略してよい）。コミットメッセージは `Sprint N attempt M: evaluator verdict <PASS|FAIL>` の形にする。
8. 判定に応じて分岐する:
   - PASS → Step 2に戻り、次のスプリントを処理する。
   - FAIL かつ `attempt < 3` → Step 3の2に戻り、同じスプリントをgeneratorに差し戻す。
   - FAIL かつ `attempt == 3` → Step 4へ。

## Step 4: 停止してユーザーに報告

1. Agentツールで `subagent_type: "agent-pipeline:advisor"` を起動し、相談の種類 `root-cause` と対象スプリント番号を渡す。
2. 3回連続で不合格になったスプリントについて、以下をユーザーに報告し、続行方針を確認する:
   - スプリント番号と目的
   - 3回分のevaluatorの指摘内容の要約
   - advisorの原因の切り分け（実装 / 仕様 / 環境）と推奨。`ADVISOR_UNAVAILABLE` の場合はその旨を書く
   - 提案する選択肢（同じ基準でもう一度試す / このスプリントをスキップして次に進める / 仕様を見直す）

ユーザーの指示を待ち、指示があればそれに従って処理を続ける。

## 全体を通して守ること

- 各スプリントのGenerator・Evaluatorは1回の試行につき1回のAgent呼び出しとする（不要な多重起動をしない）。
- advisorを呼ぶのはStep 1.5・Step 3の6・Step 4の3か所だけ。それ以外の場面では呼ばない。
- advisorの返答は助言であり、合否はevaluatorの判定だけで決める。advisorの返答を理由に判定を変えない。
- `docs/sprints/sprint-NN.md` の「評価結果」「修正方針の助言」セクションとfrontmatterの`status`以外は書き換えない（実装内容・自己評価はgeneratorが書いたものをそのまま残す）。
- 進捗はこのコマンドを再度実行すれば、`docs/sprints/*.md` のfrontmatterから自動的に再開される。
- Playwright MCPが `⏸ Pending approval` のままだとevaluatorが動かない。`claude mcp list` で `playwright` が `✔ Connected` になっているか確認するよう、必要ならユーザーに伝える。
