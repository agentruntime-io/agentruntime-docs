# llm step results

Runtime stores LLM output on the completed step as a **table**, not as parsed JSON.

## Shape

```json
{
  "model": "gpt-4o-mini",
  "text": "<assistant message content>",
  "tokens_in": 0,
  "tokens_out": 0,
  "latency_ms": 0
}
```

## Templates

```text
{{steps.<step_id>.result.text}}
```

## Lua

```lua
local llm = steps["<step_id>"].result
local raw = type(llm) == "table" and llm.text or nil
if type(raw) ~= "string" or raw == "" then error("missing LLM text") end
```

Workflow Lua has **no JSON library** (`json.decode` is nil). Do not plan an `llm` step followed by a Lua decode. When downstream steps need structured fields, use an `agent` step whose agent declares an `output_schema`: the reply is validated against it (one repair round), and each field is on the step result, e.g. `{{steps.<step_id>.result.<field>}}`. Use `llm` steps for free text (summaries, drafts) that later steps pass on as text.

Platform MCP: **`workflows_llm_step_contract`** returns the same contract inline.

## Dry-run

Downstream Lua that reads LLM output needs synthetic data:

```json
{
  "mock_step_outputs": {
    "<step_id>": { "text": "{\"people\":[]}" }
  }
}
```

Without this, validate may warn `lua_fixture_runtime` on steps that depend on LLM output.

## Credentials

`model_ref` values like `direct.openai.gpt-4o-mini` require tenant LLM keys (`llm_provider_keys_catalog`). Dry-run does not consume credits; live runs do.
