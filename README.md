# agent-skills-eval

SDK and CLI for evaluating `agentskills.io`-style skills.

This package was split from Bench AI's Skills Eval implementation so skill authors can build custom evaluators without depending on the Bench AI web app or suite engine.

## Install

```sh
npm install agent-skills-eval
```

## CLI

```sh
OPENAI_BASE_URL=https://api.openai.com/v1 OPENAI_API_KEY=... \
  npx agent-skills-eval ./skills \
  --target gpt-4o-mini \
  --judge gpt-4o-mini \
  --workspace ./agent-skills-workspace
```

Or use a YAML config:

```sh
OPENAI_API_KEY=... npx agent-skills-eval --config agent-skills-eval.yaml
```

```yaml
root: ./skills
workspace: ./agent-skills-workspace
baseline: true
target: gpt-4o-mini
judge: gpt-4o-mini
baseUrl: https://api.openai.com/v1
apiKeyEnv: OPENAI_API_KEY
include:
  - "skills/**"
exclude:
  - "**/draft-*"
concurrency: 4
layout: iteration
strict: true
report:
  enabled: true
  title: Agent Skills Report
logging:
  format: pretty # pretty, jsonl, or silent
  verbose: false
  color: auto
targetParams:
  temperature: 0
judgeParams:
  temperature: 0
```

Useful flags:

- `--baseline`: run `with_skill` and `without_skill` modes.
- `--include <glob>` / `--exclude <glob>`: filter discovered skill paths.
- `--concurrency <n>`: run eval cases in parallel. Defaults to `4`.
- `--layout iteration`: write the official agentskills.io `iteration-N` workspace layout. This is the CLI default.
- `--strict`: validate `SKILL.md` frontmatter against the agentskills.io specification before running.
- `--log-format pretty|jsonl|silent`: choose human logs, machine-readable event logs, or no event logs.
- `--log-file <path>`: write JSONL event logs to a file.
- `--no-report`: skip the static HTML report.

## SDK

```ts
import {
  OpenAICompatibleProvider,
  consoleReporter,
  evaluateSkills,
  loadConfigFile,
} from "agent-skills-eval";

const target = new OpenAICompatibleProvider({
  baseUrl: "https://api.openai.com/v1",
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
  providerName: "openai",
});

const result = await evaluateSkills({
  root: "./skills",
  workspace: "./agent-skills-workspace",
  baseline: true,
  concurrency: 4,
  workspaceLayout: "iteration",
  strict: true,
  target: { model: target.model, provider: target },
  judge: { model: target.model, provider: target },
  onEvent: consoleReporter(),
});

console.log(result);
```

You can provide any model backend by implementing the exported `Provider` interface.

Config files can also be loaded programmatically:

```ts
import { loadConfigFile } from "agent-skills-eval";

const config = loadConfigFile("./agent-skills-eval.yaml");
```

## Skill Layout

```text
my-skill/
  SKILL.md
  references/
    notes.md
  scripts/
    helper.sh
  evals/
    evals.json
    files/
      input.csv
```

`evals/evals.json`:

```json
{
  "evals": [
    {
      "id": "basic",
      "name": "basic behavior",
      "prompt": "Use the attached data to summarize revenue.",
      "files": ["evals/files/input.csv"],
      "assertions": [
        "The output identifies the highest revenue month."
      ]
    }
  ]
}
```

If `assertions` are omitted but `expected_output` is present, the SDK turns the expected output into a judge assertion so minimal agentskills.io eval files still produce pass/fail grading.

The evaluator writes portable artifacts under the workspace: `meta.json`, per-eval `prompts.json`, `grading.json`, `timing.json`, output text, optional `tool_calls.json`, `benchmark.json`, and a static HTML report.

## Logging And Reports

`consoleReporter()` emits buffered human-readable logs, so concurrent eval output stays grouped per eval case.

`jsonlReporter()` emits one JSON object per event with an ISO timestamp. Use this for CI, dashboards, or custom progress UIs.

The HTML report is static and self-contained. It summarizes pass rate, timing, tokens, assertions, prompts, outputs, and tool calls from the artifacts on disk.

## agentskills.io Compatibility

Supported:

- `SKILL.md` YAML frontmatter with required `name` and `description`.
- Optional `license`, `compatibility`, `metadata`, and `allowed-tools` frontmatter fields.
- Strict validation for name length, lowercase hyphenated name format, parent directory match, description length, and compatibility length.
- Optional `scripts/`, `references/`, and `assets/` directories. Markdown references are included in the skill context; scripts are exposed by manifest by default.
- `evals/evals.json` with `skill_name`, `evals[].id`, `prompt`, `expected_output`, `files`, and `assertions`.
- Official eval artifacts: `iteration-N/<eval>/<mode>/outputs`, `timing.json`, `grading.json`, and `benchmark.json`.
- Baseline comparison via `with_skill` and `without_skill`.

SDK extensions:

- `defaults`, model `params`, tool definitions, and deterministic `tool_assertions`.
- `workspaceLayout: "flat"` for multi-skill dashboards and report generation.
