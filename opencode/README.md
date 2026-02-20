# opencode config


## symlink(ubuntu)
```bash
ln -s ~/Dev/Tools/prompts/opencode/AGENTS.md  ~/.config/opencode
ln -s ~/Dev/Tools/prompts/opencode/opencode.json ~/.config/opencode
```
## opencode-sync-prompts(~/.local/bin/opencode-sync-prompts)

promptsファイルからopencode.jsonのpromptを更新するシェルスクリプト

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG="${HOME}/.config/opencode/opencode.json"
PROMPTS_DIR="${HOME}/.config/opencode/prompts"

tmp="$(mktemp)"

jq \
  --arg plan_prompt "$(cat "${PROMPTS_DIR}/plan.md")" \
  --arg implement_prompt "$(cat "${PROMPTS_DIR}/implement.md")" \
  --arg test_prompt "$(cat "${PROMPTS_DIR}/test.md")" \
  --arg review_prompt "$(cat "${PROMPTS_DIR}/code_review.md")" \
  '
  .agent.plan.prompt = $plan_prompt
  | .agent.implement.prompt = $implement_prompt
  | .agent.test.prompt = $test_prompt
  | .agent.code_review.prompt = $review_prompt
  ' \
  "${CONFIG}" > "${tmp}"

mv "${tmp}" "${CONFIG}"
echo "Synced prompts into ${CONFIG}"

```

spec_design (仕様設計)
役割: 仕様の設計者。計画エージェントのために正確なコンテキスト（要件、インターフェース、エッジケース、受け入れ基準など）を準備します。
特徴: 自身ではMarkdownファイルの作成・編集を行わず、ドキュメント化はすべて summary エージェントに委譲します。
plan (実装計画)
役割: 実装のプランナー。コードの記述は行わず、実装計画とテスト計画の作成に専念します。
特徴: summary エージェントに指示を出して、プロジェクトルートに agent-todo.md（チェックリスト形式のタスク一覧）を作成させます。
implement (実装)
役割: ソフトウェアの実装者。agent-todo.md の「Implement Plan（実装計画）」に基づいて実際のコーディングを行います。
特徴: Markdownファイルの編集権限を持たないため、タスク完了時のチェックリストの更新は summary エージェントに依頼します。
test (テスト)
役割: テストエンジニア (SDET)。agent-todo.md の「Test Plan（テスト計画）」に基づいてテストを追加・実行し、結果をまとめます。
特徴: 高速な単体テストを優先します。Markdownの更新やテスト結果の書き込みは summary エージェントに委譲します。
code_review (コードレビュー)
役割: 厳格なコードレビュアー。agent-todo.md の要件が満たされているか、コードの品質（保守性やセキュリティ）に問題がないかを確認します。
特徴: コードやMarkdownの編集は行いません。すべて問題なければ、summary エージェントに agent-todo.md の削除を指示します。
websearch (ウェブ検索)
役割: ウェブリサーチャー（サブエージェント）。他のエージェントからウェブ検索の要求があった場合のみ使用されます。
特徴: 一次情報源を検索し、短い回答、引用元URL、抽出した事実を返します。コードの記述は行いません。
summary (ドキュメント管理・要約)
役割: リポジトリのドキュメント編集者（サブエージェント）。
特徴: 唯一Markdownファイル（AGENTS.md、agent-todo.md、レポートなど）の作成・編集・削除が許可されているエージェントです。他のエージェントからの指示に従い、チェックリストの更新や要約の追記などを正確に行います。
