# jev-mode

A lightweight, pure-Markdown LLM skill that turns a conversational chat model into a fast, low-token, low-thinking typed decision engine inspired by [TypeSafe AI](https://typesafe.ai/)'s [Jev](https://en.wikipedia.org/wiki/Jev_(AI_model)) System One model.

Instead of emitting conversational prose or long chain-of-thought explanations, `jev-mode` evaluates a shared `state` (or a batch `states` array) against atomic `noul`, `choice`, and `score` primitives in a single pass and returns minified, machine-readable JSON with fail-closed routing gates.

<img width="921" height="603" alt="image" src="https://github.com/user-attachments/assets/bfbf48c6-2ccc-433a-81b5-a00324de8e67" />


## Why `jev-mode`?

Native [Jev](https://docs.typesafe.ai/introduction) is a discriminative, non-generative System One model built for high-volume software decisions—returning typed choices, scores, and calibrated probabilities at 70–500 ms latency and $0.042/M input tokens ($0 output) ([Wikipedia](https://en.wikipedia.org/wiki/Jev_(AI_model)); [TypeSafe Models Docs](https://docs.typesafe.ai/models)).

A prompt-level skill cannot change a chat LLM's underlying autoregressive architecture, but it *can* bring Jev's core interface and token discipline into everyday chat and agent workflows:

| Dimension | Native Jev Model | Chat LLM + `jev-mode` Skill |
| :--- | :--- | :--- |
| **Output contract** | Zero prose; typed JSON distributions ([TypeSafe Docs](https://docs.typesafe.ai/introduction)) | Minified single-line JSON (`{"answers":{...}}` or `format=tiny`) with zero conversational fluff |
| **Reasoning overhead** | Non-generative single pass ([LangChain Blog](https://www.langchain.com/blog/building-a-harness-with-jev)) | System-1 prompt rules suppress backtracking loops ([NoWait](https://arxiv.org/html/2506.08343v1)), cap scratchpad to `<= 5 words` per question ([Chain of Draft](https://redis.io/blog/token-budget-aware-llm-reasoning/)), and use `0.05`-step probabilities |
| **Batch evaluation** | Parallel question evaluation per request ([LangChain Blog](https://www.langchain.com/blog/building-a-harness-with-jev)) | Single-turn multi-question and multi-item `states: [...]` batching |
| **Calibration & latency** | RLCD-trained calibration; 70–500 ms latency ([Wikipedia](https://en.wikipedia.org/wiki/Jev_(AI_model))) | Prompt-anchored LLM estimates; hard zero-thinking caps still require API-level `reasoning_effort` / thinking-budget controls ([OpenAI Reasoning Guide](https://developers.openai.com/api/docs/guides/reasoning)) |

## Key Features

- **System-1 Reasoning Compression**: Instructs the model to decide on first read, suppress self-reflection loops ("Wait", "Hmm", "Let me double-check") ([arXiv 2506.08343](https://arxiv.org/html/2506.08343v1)), respect a tight reasoning budget ([TALE, ACL 2025](https://aclanthology.org/2025.findings-acl.1274/)), and avoid in-head floating-point `MAD` arithmetic.
- **Three Atomic Primitives**:
  - `noul`: Yes/no probability `P(true)` in `0.05` increments (`0.50` = uncertainty).
  - `choice`: Unordered classification (2 to 255 options; caps to `top_k: 3` + `"_other"` when `n > 4`).
  - `score`: Ordered rubric (2 to 10 levels) returning `level`, `probabilities`, and top-2 `margin`.
- **Fail-Closed Gating**:
  - Symmetric `noul` gating using `c = max(noul, 1 - noul)` so both confident-true (`0.90`) and confident-false (`0.05`) proceed, while `0.40–0.60` abstains.
  - Scale-invariant `choice` / `score` gating using `margin = p1 - p2` (`< 0.15` abstain, `0.15–0.29` review, `>= 0.30` proceed).
- **Prompt-Injection Sandboxing**: Enforces a strict `Data-Not-Instructions` boundary so adversarial directives embedded inside untrusted `state` payloads cannot hijack routing or gate outputs.
- **Task-Mode & Hybrid Support**: Supports `states: [...]` batch triage, `format=tiny` scalar output, `explain=true` (15-word rationale), `gate_then_act=true` (run downstream actions only when gates return `"proceed"`), and `exit jev-mode`.

## Repository Structure

```text
jev-mode/
├── README.md   # Documentation and usage examples
└── SKILL.md    # Self-contained skill definition (YAML frontmatter + instructions)
```

## Installation

1. Upload `SKILL.md` (or this `jev-mode/` folder) into your agent platform's Skills Manager.
2. Confirm the skill is enabled as `jev-mode`.
3. Trigger it in any chat or task-mode session using phrases like `Enable jev-mode`, `Run in Jev mode`, or `@jev-mode`.
4. Return to normal conversational prose at any time by saying `exit jev-mode` or `normal mode`.

## Usage Examples

### 1. Single-State Decision (Default Minified JSON)

**Prompt:**
```text
Enable jev-mode. State: "My corporate wire transfer failed twice today and our settlement deadline is in 2 hours." Decide is_fraud (noul), route (billing, technical, general), and severity (low, medium, high, critical).
```

**Output:**
```json
{"answers":{"is_fraud":{"type":"noul","noul":0.05,"gate":"proceed"},"route":{"type":"choice","choice":"billing","probabilities":{"billing":0.85,"technical":0.10,"general":0.05},"margin":0.75,"gate":"proceed"},"severity":{"type":"score","level":"high","probabilities":[0.05,0.15,0.45,0.35],"margin":0.10,"gate":"abstain"}}}
```

### 2. Scalar-Only Output (`format=tiny`)

**Prompt:**
```text
jev-mode format=tiny. State: "Routine monthly statement PDF downloaded cleanly." Decide is_urgent (noul), route (billing, technical, general), severity (low, medium, high).
```

**Output:**
```json
{"is_urgent":0.05,"route":"general","severity":"low"}
```

### 3. Multi-Item Batch Mode (`states: [...]`)

**Prompt:**
```text
jev-mode batch. Questions: is_urgent (noul).
States:
1. "Production checkout returning 500s for all users."
2. "Maybe check that thing from earlier if needed."
```

**Output:**
```json
{"batch":[{"id":1,"answers":{"is_urgent":{"type":"noul","noul":0.95,"gate":"proceed"}}},{"id":2,"answers":{"is_urgent":{"type":"noul","noul":0.45,"gate":"abstain"}}}]}
```

### 4. Task-Mode Gate-Then-Act (`gate_then_act=true`)

```text
Use jev-mode with gate_then_act=true: evaluate whether this draft email has any compliance risk (noul) or missing key facts (noul). If both gates proceed with no risk, save it as a Gmail draft.
```

## Limitations

- **Not true Jev inference**: `jev-mode` optimizes prompt/output contracts on generative LLMs; it does not provide RLCD-trained probability calibration or native discriminative latency.
- **API-level thinking controls**: Prompt directives reduce internal reasoning, but strict zero-thinking caps on reasoning models require API parameters (`reasoning_effort` or thinking-budget settings) ([OpenAI Reasoning Guide](https://developers.openai.com/api/docs/guides/reasoning)).
- **No exact arithmetic or date logic**: Keep fuzzy semantic classification in `jev-mode` and delegate exact calculations, date math, and `exact_math=true` Jev formulas to deterministic code.
