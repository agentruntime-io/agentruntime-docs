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

When the prompt asks for JSON, decode it in a **follow-on** `lua` step with the built-in `json` library, and guard against invalid output:

```lua
local ok, parsed = pcall(json.decode, raw)
if not ok or type(parsed) ~= "table" then error("LLM text is not a JSON object") end
return { extraction = parsed }
```

`json.decode`: objects/arrays become tables, numbers are doubles, `null` becomes nil (a hole in arrays), invalid JSON raises an error. `json.encode`: keys 1..n become an array, other tables an object, empty table `{}`. Max 4 MiB per call.

When downstream steps must rely on the fields, prefer an `agent` step whose agent declares an `output_schema`: the reply is validated against it (one repair round), and each field is on the step result, e.g. `{{steps.<step_id>.result.<field>}}`.

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

`model_ref` values like `direct.openai.gpt-4o-mini` need a model account: the platform's, or a workspace connection switched on for models (`llm_provider_keys_catalog` lists both). Dry-run does not consume credits; live runs do.
