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
