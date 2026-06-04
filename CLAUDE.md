# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## Overview

`@stackone/defender` is a prompt injection defense framework for AI tool-calling. It detects and
neutralizes indirect prompt injection attacks hidden in tool results (emails, documents, PRs, etc.)
before they reach the LLM, via a two-tier pipeline: Tier 1 regex pattern detection/sanitization (sync,
~1ms) and Tier 2 ML classification using a fine-tuned MiniLM ONNX model (~22MB, bundled, CPU-only).
Published to npm as a library (ESM + CJS).

## Commands

Package manager: **npm** (`package-lock.json`). Use only npm — do not introduce bun/pnpm/yarn.

- `npm install` — install dependencies.
- `npm run build` — production build via tsdown (minified) + copy bundled models to `dist/`.
- `npm run build:dev` — development (unminified) build + copy models.
- `npm test` — run the Vitest test suite once.
- `npm run test:watch` — Vitest in watch mode.
- `npm run test:typecheck` — typecheck the tests (`tsc --noEmit -p tsconfig.tests.json`).
- `npm run lint` — Biome check (`code:check`) over `src/`.
- `npm run lint:fix` — Biome check with `--write` (autofix).
- `npm run code:format` / `code:format:fix` — Biome formatting.
- `npm run clean` — remove `dist/`.

ML-model dependencies (`@huggingface/transformers`, `onnxruntime-node`, `fasttext.wasm`) are optional
peer deps; they're present as devDependencies here for building/testing.

## Architecture

Library entry point is `src/index.ts`, which re-exports the public API. Source under `src/`:

- `core/prompt-defense.ts` — main API: `createPromptDefense()` / `PromptDefense` class. Methods
  `defendToolResult(value, toolName)`, `defendToolResults(items)` (batch), `analyze(text)`,
  `warmupTier2()`. Returns `DefenseResult`. Orchestrates Tier 1 + Tier 2.
- `core/tool-result-sanitizer.ts` — applies sanitization across a tool result's risky fields.
- `classifiers/` — detection layers:
  - `pattern-detector.ts` + `patterns.ts` — Tier 1 regex injection-pattern detection.
  - `tier2-classifier.ts` + `onnx-classifier.ts` — Tier 2 ML classifier (sentence-level scoring).
  - `models/minilm-multihead-v5/` — the bundled int8-quantized ONNX model + tokenizer/config (current
    default; calibration encoded in `classifier_config.json`).
- `sanitizers/` — Tier 1 transforms: `normalizer` (Unicode/homoglyph), `role-stripper`,
  `pattern-remover`, `encoding-detector` (Base64/URL), `leet-normalizer`, composed via `sanitizer.ts`.
- `utils/` — `field-detection.ts` (per-tool risky-field selection), `structure.ts`,
  `boundary.ts` (`generateBoundaryInstructions`, `containsBoundaryPatterns` for opt-in boundary tags).
- `sfe/` — optional FastText (`model.ftz`) preprocessor (`preprocess.ts`); off by default (`useSfe`).
- `types.ts` / `config.ts` — shared types (`RiskLevel`, `Tier1Result`) and defaults.

Tests live in `specs/**/*.spec.ts` (Vitest, globals enabled). Build is tsdown (config in
`tsdown.config.ts`); `scripts/copy-models.cjs` mirrors `src/classifiers/models/<name>` →
`dist/models/<name>` and `src/sfe/model.ftz` → `dist/sfe/` after each build — Tier 2 resolves models by
path relative to the compiled `dist/` file, so this copy step is required.

## Conventions

- Formatting/linting via **Biome** (`biome.json`): tabs, indent width 4, line width 120. Imports are
  auto-organized. Models dir is excluded from Biome.
- Releases are automated via **release-please** (`release-please-config.json`,
  `.release-please-manifest.json`); commits should follow Conventional Commits (a semantic-PR check runs
  in CI). CI workflows live in `.github/workflows/`.
- `allowed` (gated by `blockHighRisk`) is the block/allow signal; `riskLevel` is diagnostic-only and is
  only ever escalated, never reduced.
