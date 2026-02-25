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

if [[ ! -f "${CONFIG}" ]]; then
  echo "Config not found: ${CONFIG}" >&2
  exit 1
fi

if [[ ! -d "${PROMPTS_DIR}" ]]; then
  echo "Prompts directory not found: ${PROMPTS_DIR}" >&2
  exit 1
fi

if ! command -v jq >/dev/null 2>&1; then
  echo "jq is required but not installed" >&2
  exit 1
fi

trap 'rm -f "${tmp}"' EXIT

# jq 引数とフィルタの組み立て
ARGS=()
FILTER='.'
updated=0

# Markdownファイルごとにプロンプトを抽出して jq 引数に追加
for f in "${PROMPTS_DIR}"/*.md; do
  [[ -f "$f" ]] || continue

  name=$(basename "$f" .md)

  if ! jq -e --arg name "$name" '.agent[$name] != null' "${CONFIG}" >/dev/null; then
    echo "Skip unknown agent markdown: ${f}" >&2
    continue
  fi
  
  # ## Prompt 以降を抽出し、先頭・末尾の空行を削除
  content=$(awk '
    BEGIN {in_prompt=0}
    /^## Prompt[[:space:]]*$/ {in_prompt=1; next}
    in_prompt {print}
  ' "$f")

  content=$(printf '%s\n' "$content" | sed '/./,$!d' | tac | sed '/./,$!d' | tac)

  if [[ -z "$content" ]]; then
    echo "Skip empty prompt body: ${f}" >&2
    continue
  fi
  
  ARGS+=(--arg "p_${name}" "$content")
  FILTER+=" | .agent[\"${name}\"].prompt = \$p_${name}"
  updated=$((updated + 1))
done

if [[ "$updated" -eq 0 ]]; then
  echo "No prompts were updated. Ensure markdown files contain a '## Prompt' section and match agent keys." >&2
  exit 1
fi

# jq を一回だけ実行して更新
jq "${ARGS[@]}" "$FILTER" "${CONFIG}" > "${tmp}"

mv "${tmp}" "${CONFIG}"
echo "Synced all prompts from ${PROMPTS_DIR} into ${CONFIG}"
```

## エージェント構成

### 1. 仕様策定と計画フェーズ（メイン：spec）
- **spec (Primary)**: 仕様策定、全体計画（高推論）。ユーザーの依頼を実行可能な計画に落とし込む。
- **explore (Subagent)**: コードベース調査。読み取り専用で現在のコードベースを調査する。
- **internet_research (Subagent)**: インターネット検索・調査。知識の欠落を補うための外部リサーチを行う。
- **draft_planner (Subagent)**: ドラフト計画の作成。`.agents/plans/` 内にドラフト計画を作成する。
- **plan_reviewer (Subagent)**: 計画書の厳格な査読（高推論）。最終計画およびテスト仕様書の厳格な査読を行う。

### 2. 実装オーケストレーションフェーズ（メイン：orchestrator）
- **orchestrator (Primary)**: タスク分割、実行指示（司令塔）。計画を小さなタスクに分解し、サブエージェントに委譲する。
- **general (Subagent)**: 調査を伴うコード実装。委譲されたタスクをエンドツーエンドで実行する。
- **implement (Subagent)**: 局所的なコード編集。指示が明確な箇所へのピンポイントなパッチ適用を行う。
- **debugger (Subagent)**: バグの原因究明（高精度）。バグの調査、再現、根本原因の分析を行う。
- **test_designer (Subagent)**: テスト仕様の設計（高精度）。機能変更に合わせてテストの仕様書（test-spec）を作成する。

### 3. 検証と監査フェーズ
- **tester (Subagent)**: テスト実行と失敗報告。テストを実行し、失敗した場合は failure-report を作成して報告する。
- **code_reviewer (Subagent)**: コードの厳格な査読（高推論）。変更されたコードの厳格な査読を行う。
- **doc_auditor (Subagent)**: ドキュメントの乖離チェック。実装とドキュメントにズレがないかチェックし、更新指示書を作成する。

## ワークフローと「関所」

このフローにはAIが暴走しないための「関所」が設けられています。

1. **初期調査**: `spec` が開始され、必要に応じて `explore` を呼び出して、現在のコードベースを調査します。
2. **仕様の明確化 (Hard Gate)**: 曖昧な点がある場合、`spec` はユーザーに質問し、不明点を解消します。これが解決するまで次へ進めない「ハードゲート」になっています。
3. **外部リサーチ**: 知識に欠落がある場合、`internet_research` を呼び出して事実を確認します。
4. **ドラフト計画の作成**: `spec` が `draft_planner` に指示し、`.agents/plans/` 内に「ドラフト（下書き）」を作成させます。
5. **ユーザー承認 (Draft Confirmation Gate)**: **[重要]** ユーザーがドラフトを確認し、明示的に「承認」するまで実装には進みません。
6. **最終計画の作成と査読**: 承認されたドラフトを元に `spec` が最終計画を作成し、`plan_reviewer` がその整合性を厳格にチェックします。
7. **タスク分割と並行実装**: `orchestrator` が最終計画を小さなタスク単位に分解し、`general`, `implement`, `debugger` などのサブエージェントに作業を振ります。
8. **テスト設計**: 機能変更に合わせて、`test_designer` がテストの仕様書（test-spec）を作成します。
9. **検証と監査**: 実装完了後、`tester` にテストを実行させ、`code_reviewer` にコードレビューを依頼し、`doc_auditor` にドキュメント監査を依頼します。
