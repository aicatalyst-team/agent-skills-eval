# PoC Plan: agent-skills-eval

## Project Classification
- **Type:** llm-app
- **Key Technologies:** TypeScript, Node.js, Commander.js, js-yaml, OpenAI-compatible LLM APIs
- **ODH Relevance:** This is an LLM evaluation/benchmarking CLI tool that tests AI agent skills. It's relevant to Open Data Hub as a model evaluation and quality assurance tool — it can be used to evaluate the quality of models served through ODH's model serving infrastructure (KServe, ModelMesh) by running skill evaluations against OpenAI-compatible endpoints those servers expose.

## PoC Objectives
What we want to prove:
1. The TypeScript project compiles cleanly and produces a working CLI binary inside a container
2. The CLI tool starts correctly and presents its help/version information
3. The bundled example skill structure is included and discoverable by the tool
4. The compiled dist/ output contains all expected module entry points (cli, index, provider, config, reporters)

## Infrastructure Requirements
- **Inference Server:** none (this tool *calls* inference servers; it doesn't serve models itself)
- **Vector Database:** none
- **Embedding Model:** none
- **GPU Required:** no
- **Persistent Storage:** none (evaluation artifacts are ephemeral for PoC)
- **Resource Profile:** small (256Mi RAM, 250m CPU — this is a lightweight Node.js CLI)
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: help-output
- **Description:** Verify the CLI shows help text with available commands and options
- **Type:** cli
- **Input:** `node dist/cli.js --help`
- **Expected:** Job exits 0, outputs usage information listing key options such as `--target`, `--judge`, `--baseline`, `--strict`
- **Timeout:** 15 seconds

### Scenario 2: version-check
- **Description:** Verify the CLI reports its version
- **Type:** cli
- **Input:** `node dist/cli.js --version`
- **Expected:** Job exits 0, outputs version string `0.1.1`
- **Timeout:** 10 seconds

### Scenario 3: discover-example-skill
- **Description:** Run the CLI against the bundled example skill directory to verify skill discovery and YAML parsing logic. No API key is provided, so we expect discovery to work but the actual evaluation to fail gracefully.
- **Type:** cli
- **Input:** `node dist/cli.js ./examples --target gpt-4o-mini --judge gpt-4o-mini --dry-run 2>&1 || true`
- **Expected:** Output demonstrates the CLI parsed the skill directory structure and identified the example skill. May exit non-zero due to missing API key, but the discovery/validation phase should produce meaningful output.
- **Timeout:** 30 seconds

### Scenario 4: build-integrity
- **Description:** Verify that the TypeScript build produced all expected compiled JavaScript files in dist/
- **Type:** cli
- **Input:** `ls -la dist/cli.js dist/index.js dist/provider.js dist/config.js`
- **Expected:** Job exits 0, listing all expected compiled JavaScript files confirming the build was successful
- **Timeout:** 10 seconds

## Dockerfile Considerations

This is a **CLI tool**, not a server. The Dockerfile should:

- Use a Node.js 18+ base image (e.g., `node:18-alpine` or `node:20-alpine`)
- Copy `package.json` and `package-lock.json`, run `npm ci` to install dependencies
- Copy the `src/`, `tsconfig.json`, and `examples/` directories
- Run `npm run build` to compile TypeScript to JavaScript in `dist/`
- The compiled output lives in `dist/` — the final image needs `dist/`, `node_modules/`, `examples/`, and `package.json`
- Consider a multi-stage build: build stage compiles TypeScript, production stage only includes `dist/`, production `node_modules` (only `commander` and `js-yaml`), and `examples/`
- **ENTRYPOINT** should be `["node", "dist/cli.js"]`
- **CMD** should default to `["--help"]`
- **Do NOT add EXPOSE** — there is no port to expose. This tool does not listen on any port.
- The tool has only 2 runtime dependencies (`commander`, `js-yaml`), so the image can be very lean

## Deployment Considerations

- **Do NOT deploy as a Deployment** — this is a CLI tool that runs a command and exits. Deploying it as a Deployment would cause CrashLoopBackOff since the process terminates after execution.
- **Do NOT create a Service** — there is no port to expose. The tool does not listen for incoming connections.
- **Deploy as a Kubernetes Job** for each test scenario. Each Job runs a specific CLI command, and success is verified by:
  1. Checking the Job's exit code (should be 0 for most scenarios)
  2. Inspecting the Job's logs via `kubectl logs`
- The `examples/` directory should be included in the container image so the skill discovery test can find the bundled example skill
- No environment variables are strictly required for the PoC scenarios (the tool needs `OPENAI_API_KEY` for actual evaluations, but our PoC tests focus on CLI functionality, build integrity, and skill discovery — not live API calls)
- If a future PoC iteration wants to test actual evaluation runs against an ODH-served model, set `OPENAI_API_KEY` and `OPENAI_BASE_URL` to point to the ODH model serving endpoint