# Popper-Style Reactive Dataflow Pipeline for Automated Program Repair

## 1. Goal

Design a dataflow pipeline using Popper's reactive dataflow model to automatically repair Java bugs from the Defects4J dataset. The system takes a **project source directory** (not individual bug checkouts), iterates through bugs, attempts fixes using LLM-generated patches, and records results.

Dataset: **Defects4J** — real Java bugs where code compiles but fails at runtime, with a known triggering test and ground-truth fix.

---

## 2. Reference Implementations

### 2.1 Claude Code (at `/home/adnan/Downloads/src`)

Claude Code uses an **implicit self-orchestrated approach** within a single LLM call per turn:

| Aspect | Claude Code |
|---|---|
| **Context assembly** | `buildSystemPromptBlocks()` constructs system prompt from identity + instructions + tools + user context |
| **Tool system** | Each tool (FileEditTool, Bash, Grep, etc.) is a typed function the LLM can invoke; tools have descriptions, parameters, and guards (read-before-edit, staleness) |
| **Edit mechanism** | Exact `old_string` match replacement; `read` before `edit` enforced; staleness check via fingerprint |
| **Loop** | Agentic — LLM decides next action each turn within a single session |
| **Structure** | No structured context separation; everything (system prompt + conversation history + tool results) is concatenated into one context window |
| **Relevance** | Shows how prompt assembly works, how tools are defined, and how edits are applied — useful for our PromptBuilder and PatchApplier operators |

Key files referenced:
- `constants/prompts.ts` — system prompt assembly (~300 lines)
- `services/api/claude.ts` — `queryModelWithStreaming()` (line 752), `buildSystemPromptBlocks()` (line 3213)
- `query.ts` — query loop, `deps.callModel()` (line 659)
- `tools/FileEditTool/FileEditTool.ts` — edit tool with old_string match, read-before-edit guard, staleness check

### 2.2 Popper Paper ("POPPER: A Dataflow System for In-Flight Error Handling in Machine Learning Workflows")

Popper is a **reactive dataflow** system with explicit error handling operators. Key concepts:

| Concept | Meaning |
|---|---|
| **Operator** | Compute node with typed input/output ports |
| **Row** | Immutable data record flowing through edges |
| **∆⁺** | Append — new rows added to a stream |
| **∆⁻** | Delete — existing rows removed from a stream |
| **im+** | Incremental monotonic (append only) |
| **im±** | Incremental non-monotonic (append + delete) |
| **i** | Non-incremental (full recompute downstream) |
| **Forward edge** | Data flows downstream (∆⁺ + ∆⁻) |
| **Backward correction edge** | Error handler edits upstream operator's *persisted output* by emitting ∆⁻ (remove bad) + ∆⁺ (append corrected) |
| **detect()** | Error handler checks if a row is erroneous |
| **correct()** | Error handler produces the correction (∆⁻ + ∆⁺) |
| **eventually()** | When a row cannot be corrected, emit it as an uncorrectable result |

#### Key insight from the paper

> Error handlers don't orchestrate or call upstream operators. They emit ∆⁻(fail) + ∆⁺(corrected) on the *upstream operator's output stream*, which triggers downstream recomputation automatically through the dataflow engine's reactivity.

---

## 3. Pipeline Topology — Operators

### 3.1 Forward Flow

```
Checkout → TestRunner_A → ContextCollector → PromptBuilder → TokenOptimizer → LLMCaller → PatchApplier → TestRunner_B → VerdictOutputter
```

### 3.2 Operator Catalog

| Operator | Type | Input | Output | Description |
|---|---|---|---|---|
| **Defects4jCheckout** | extractor | (none) | ProjectRow | Checkout buggy source, export trigger/relevant tests |
| **TestRunner_A** | processor | ProjectRow | BugRow | Compile → run suite → parse stack trace → emit bug location (file:line). Merges FailLocator. |
| **ContextCollector** | processor | BugRow | ContextRow (D or E) | Reads files from disk on-demand. D: method + test; E: full trace slice |
| **PromptBuilder** | processor | ContextRow | PromptRow | Assemble LLM prompt using a strategy (generic, CoT, minimal-change, TDD) |
| **TokenOptimizer** | processor | PromptRow | OptPromptRow | Compress only code context portion of prompt (not system instructions) to fit token limits |
| **LLMCaller** | processor | OptPromptRow | PatchRow | Call LLM API → parse response → extract file path, old_string, new_string, reasoning |
| **PatchApplier** | processor | PatchRow | AppliedPatchRow | Apply old→new string replacement to files on disk |
| **TestRunner_B** | processor + errorHandler | AppliedPatchRow | PassRow / EventualRow | Run relevant suite; decide pass (all pass), fail (backward correct), or eventually (exhausted) |
| **VerdictOutputter** | outputter | PassRow, EventualRow | (terminal) | Emit final results: fixed patches + unfixed bugs with metrics |

### 3.3 Source lives on disk, not in rows

No SourceRow. Operators read files from disk on-demand (like Claude Code tools). Checkout emits only a lightweight `ProjectRow {project_dir, bug_id, triggering, relevant}`. ContextCollector reads source when it has the bug location. PatchApplier writes edits directly to the filesystem.

---

## 4. The Two TestRunner Instances

| Instance | Position | Purpose | Outputs |
|---|---|---|---|
| **TestRunner_A** | After Checkout | **Localization** — find what's broken | BugRow (file:line:method from stack trace) |
| **TestRunner_B** | After PatchApplier | **Validation** — did the patch fix it? | PassRow (pass port) or backward correction (fail port) or EventualRow (eventually()) |

Same operator class, different graph positions. TestRunner_A is a pure processor. TestRunner_B is a processor *and* error handler (has backward edges).

FailLocator is **merged into TestRunner_A** — stack → file:line parsing is a trivial ~5 line filter (drop framework frames, take first project-owned frame). Not worth a separate node.

---

## 5. Backward Correction Model

### 5.1 Approach: Popper-style backward edges (NOT forward cycles)

We adopted the **Popper backward correction model** over forward cycles.

**Why backward corrections?**

- A forward cycle (feedback from TestRunner_B back to an upstream operator via a graph edge) requires operators to maintain state across iterations — they'd need to know "am I being re-invoked with a new conjecture?"
- Popper's model is cleaner: TestRunner_B detects failure, then emits **∆⁻ + ∆⁺** on the upstream operator's *output stream*. The upstream operator's output is a persisted row set. TestRunner_B deletes the old row and appends a corrected one. Reactivity propagates the change forward automatically — no special loop logic needed in any operator.
- No operator needs to be "loop-aware". Every operator just processes its input rows.

### 5.2 What backward correction means in practice

When TestRunner_B detects a test failure:

1. It selects the next conjecture (e.g., escalate context D→E)
2. It emits **∆⁻(old ContextRow)** + **∆⁺(new ContextRow)** on ContextCollector's output stream
3. ContextCollector's downstream — PromptBuilder — sees a new row and reactively recomputes
4. The new prompt flows through TokenOptimizer → LLMCaller → PatchApplier → TestRunner_B
5. TestRunner_B evaluates the new patch

No operator code changes between iterations. The dataflow engine handles propagation.

### 5.3 Backward correction targets

| Upstream Operator | What the correction changes | Example |
|---|---|---|
| **ContextCollector** | Context level escalation | D (method+test) → E (trace slice) |
| **PromptBuilder** | Prompt strategy | generic → CoT → minimal-change → TDD |
| **LLMCaller** | Model tier | light model → heavy model |
| **TokenOptimizer** | Compression aggressiveness | aggro (compress heavily) → gentle (compress less) |

### 5.4 Correlation between backward target and TestRunner_B

TestRunner_B does not have a separate error handler node. TestRunner_B itself IS the error handler — it has the signal (test pass/fail) and emits the backward correction directly.

```
TestRunner_B
  ├── pass port ───────────→ PassRow → VerdictOutputter
  ├── fail port ── backward ─→ ContextCollector (∆⁻D + ∆⁺E)
  │               ── backward ─→ PromptBuilder (∆⁻old_strat + ∆⁺new_strat)
  │               ── backward ─→ LLMCaller (∆⁻light + ∆⁺heavy)
  │               ── backward ─→ TokenOptimizer (∆⁻aggro + ∆⁺gentle)
  └── eventually() port ──→ EventualRow → VerdictOutputter
```

The `eventually()` output is reached when all 32 conjecture combinations are exhausted for a given bug. The bug is recorded as unfixed with attempt metrics.

---

## 6. Conjecture Space

### 6.1 Dimensions

| Dimension | Values | Count |
|---|---|---|
| **Context level** | D (method + test file), E (full trace slice) | 2 |
| **Model tier** | light (smol/haiku), heavy (claude/gpt4) | 2 |
| **Token compression** | aggro (compress heavily), gentle (compress lightly) | 2 |
| **Prompt strategy** | generic, chain-of-thought (CoT), minimal-change, TDD | 4 |

Total: **2 × 2 × 2 × 4 = 32**

### 6.2 Escalation sequence

Ordered by cost — cheapest first. light model + D-context + aggro before heavy + E + gentle.

| Range | Model | Context | Strategy | Compression |
|---|---|---|---|---|
| 1-4 | light | D | generic → CoT → minimal-change → TDD | aggro |
| 5-8 | light | D | generic → CoT → minimal-change → TDD | gentle |
| 9-12 | light | E | generic → CoT → minimal-change → TDD | aggro |
| 13-16 | light | E | generic → CoT → minimal-change → TDD | gentle |
| 17-20 | heavy | D | generic → CoT → minimal-change → TDD | aggro |
| 21-24 | heavy | D | generic → CoT → minimal-change → TDD | gentle |
| 25-28 | heavy | E | generic → CoT → minimal-change → TDD | aggro |
| 29-32 | heavy | E | generic → CoT → minimal-change → TDD | gentle |

Rationale: try cheap conjectures first (light model, D-context, aggro compression). If those fail, escalate to expensive ones (heavy model, E-context, gentle compression). Eventually() after all 32 exhausted.

---

## 7. Key Design Decisions

### 7.1 Sequential fix, not batch

Take the first failing test, fix it, re-run the suite, cascade-fix remaining failures. Do NOT pre-enumerate all failures upfront.

### 7.2 Single project entry point

The pipeline takes a project source directory (e.g., `Chart/`), discovers bugs within it, and processes them one by one. Not individual bug checkouts.

### 7.3 Source lives on disk — no SourceRow

Operators read/write files from disk on-demand (like Claude Code tools). Checkout emits only a lightweight `ProjectRow`. ContextCollector reads source when it has the bug location. PatchApplier writes edits directly to the FS.

### 7.4 FailLocator merged into TestRunner_A

Stack trace → file:line is a trivial filter (drop framework frames, take first project-owned frame). Not worth a separate operator.

### 7.5 Stack trace filtering

Drop framework/library frames from the stack trace. Take the first project-owned frame as the bug location.

### 7.6 No compile-error path

Defects4J bugs always compile. No compile-error recovery is needed.

### 7.7 PatchRow includes llm_reasoning

```python
@dataclass
class PatchRow:
    file_path: str
    old_str: str
    new_str: str
    model: str
    reasoning: str  # LLM's explanation for the fix
```

### 7.8 TokenOptimizer compresses only code context

System instructions (the prompt structure, instructions to the LLM) are NOT compressed — only the surrounding method code and trace slices.

### 7.9 Backward correction model (not forward cycles)

As detailed in §5.1 — the Popper approach of editing upstream operator output is cleaner than adding cycle-aware loop state to operators.

---

## 8. Row Schemas

| Row Type | Fields |
|---|---|
| **ProjectRow** | `project_dir: str, bug_id: str, triggering_tests: list[str], relevant_tests: list[str]` |
| **BugRow** | `project_dir: str, file: str, line: int, method: str, test_class: str` |
| **ContextRow** | `level: str (D|E), method_source: str, test_source: str, trace_slice: str (E only)` |
| **PromptRow** | `strategy: str, prompt_text: str` |
| **OptPromptRow** | `strategy: str, prompt_text: str, tokens_saved: int` |
| **PatchRow** | `file: str, old_str: str, new_str: str, model: str, reasoning: str` |
| **AppliedPatchRow** | `success: bool, file: str, old_str: str, new_str: str` |
| **PassRow** | `bug_id: str, patch: PatchRow, tests_passed: list[str]` |
| **EventualRow** | `bug_id: str, unfixed: bool, attempts: int, last_error: str` |

---

## 9. Unresolved Questions (for grill-me)

- Edge label semantics: which edges are `im+` vs `im±` vs `i`?
- Operator state: do operators maintain state across invocations? (e.g., does ContextCollector remember previous context level?)
- Lineage tracking: how does a PatchRow know which BugRow → ContextRow → PromptRow → OptPromptRow produced it?
- Error semantics: what happens when an operator itself crashes (not a test failure but a Python exception)?
- Dataflow engine: which Popper implementation to use as base?

---

## 10. Key Files Referenced During Design

| File | What it contains |
|---|---|
| `/home/adnan/Downloads/src/constants/prompts.ts` | Claude Code system prompt assembly |
| `/home/adnan/Downloads/src/services/api/claude.ts` | LLM API call + prompt block building |
| `/home/adnan/Downloads/src/tools/FileEditTool/FileEditTool.ts` | Edit tool with old_string match and guards |
| `/home/adnan/Downloads/src/query.ts` | Query loop with context prepending |
| `/home/adnan/Desktop/Popper/popper-demo 1.pdf` | Popper paper |
| `/home/adnan/Desktop/Popper/apr-dataflow.md` | Current Mermaid diagrams |
