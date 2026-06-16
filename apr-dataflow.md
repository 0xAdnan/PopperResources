# APR Pipeline — Popper Reactive Dataflow (Final)

## Legend

| Symbol | Meaning |
|---|---|
| `→` solid | Forward data edge |
| `⇠` dashed red | Backward correction edge (Popper error handler) |
| `im+` | Incremental monotonic (appends only) |
| `im±` | Incremental non-monotonic (appends + deletes) |
| `i` | Non-incremental (recompute downstream) |
| `∆⁺` | Append stream |
| `∆⁻` | Delete stream |

---

## Full Dataflow

```mermaid
flowchart TB
    %% === STYLE DEFINITIONS ===
    classDef extractor fill:#1a73e8,color:#fff,stroke:#0d47a1
    classDef processor fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef errorHandler fill:#c62828,color:#fff,stroke:#b71c1c
    classDef outputter fill:#4a148c,color:#fff,stroke:#311b92
    classDef tokenOp fill:#e65100,color:#fff,stroke:#bf360c
    classDef llm fill:#00695c,color:#fff,stroke:#004d40

    %% === EXTRACTOR ===
    Checkout[\"Defects4jCheckout<br/>extractor<br/><i>checkout + walk src + export tests</i>"/]:::extractor

    %% === STREAMS ===
    SrcRows[\"SourceRow+<br/>{file_path, content}"/]
    TestRows[\"TestRow+<br/>{triggering, relevant}"/]

    Checkout -->|"im+"| SrcRows
    Checkout -->|"im+"| TestRows

    %% === LOCALIZATION PATH ===
    TestRunner_A[TestRunner_A<br/>processor<br/><i>run full suite → failures</i>]:::processor
    FailLocator[FailLocator<br/>processor<br/><i>stack trace → file:line</i>]:::processor
    ContextCollect[ContextCollector<br/>processor<br/><i>D: method+test &vert; E: trace slice</i>]:::processor

    BugRow[\"BugRow<br/>{file, line, method, test}"/]
    TestResult[\"TestResultRow<br/>{failures: [{test, stack}]}"/]
    CtxRow[\"ContextRow<br/>{level, method, test, trace}"/]

    SrcRows -->|"i"| TestRunner_A
    TestRows -->|"im+"| TestRunner_A
    TestRunner_A -->|"im+"| TestResult
    TestResult -->|"im+"| FailLocator
    FailLocator -->|"im+"| BugRow
    BugRow -->|"im+"| ContextCollect
    ContextCollect -->|"im+"| CtxRow

    %% === CONJECTURE GENERATION ===
    PromptBuilder[PromptBuilder<br/>processor<br/><i>context → LLM prompt</i>]:::processor
    TokenOptimizer[TokenOptimizer<br/>processor<br/><i>compress code context</i>]:::tokenOp
    LLMCaller[LLMCaller<br/>processor<br/><i>API call → patch</i>]:::llm
    PatchApplier[PatchApplier<br/>processor<br/><i>apply old→new string</i>]:::processor
    TestRunner_B[TestRunner_B<br/>processor + errorHandler<br/><i>pass | fail | eventually</i>]:::errorHandler
    VerdictOut[\"VerdictOutputter<br/>outputter<br/><i>fixed / unfixed results</i>"/]:::outputter

    PromptRow[\"PromptRow<br/>{strategy, prompt_text}"/]
    OptimizedRow[\"OptPromptRow<br/>{compressed, tokens_saved}"/]
    PatchRow[\"PatchRow<br/>{file, old_str, new_str, model, reasoning}"/]
    AppliedRow[\"AppliedPatchRow<br/>{success, file, old_str, new_str}"/]
    PassRow[\"PassRow<br/>{bug_id, patch, tests_passed}"/]
    EventualRow[\"EventualRow<br/>{bug_id, unfixed, attempts}"/]

    CtxRow -->|"im+"| PromptBuilder
    PromptBuilder -->|"im+"| PromptRow
    PromptRow -->|"im+"| TokenOptimizer
    TokenOptimizer -->|"im+"| OptimizedRow
    OptimizedRow -->|"im+"| LLMCaller
    LLMCaller -->|"im+"| PatchRow
    PatchRow -->|"im+"| PatchApplier
    PatchApplier -->|"im+"| AppliedRow
    AppliedRow -->|"im+"| TestRunner_B

    %% === TEST RUNNER B OUTPUT PORTS ===
    TestRunner_B -->|"pass"| PassRow
    TestRunner_B -->|"eventually()"| EventualRow
    PassRow -->|"im+"| VerdictOut
    EventualRow -->|"im+"| VerdictOut

    %% === BACKWARD CORRECTIONS (Popper style: edit upstream output) ===
    TestRunner_B -.->|"backward: ∆⁻(D-ctx) + ∆⁺(E-ctx)<br/>context escalation"| ContextCollect
    TestRunner_B -.->|"backward: ∆⁻(old prompt) + ∆⁺(new prompt)<br/>strategy change"| PromptBuilder
    TestRunner_B -.->|"backward: ∆⁻(light) + ∆⁺(heavy)<br/>model upgrade"| LLMCaller
    TestRunner_B -.->|"backward: ∆⁻(aggro) + ∆⁺(gentle)<br/>reduce compression"| TokenOptimizer
```

---

## Correction Lifecycle

```mermaid
stateDiagram-v2
    state "Conjecture (strategy, model, context, compression)" as CONJ
    
    [*] --> CONJ: PromptBuilder emits
    CONJ --> Running: flows forward through pipeline
    Running --> Pass: TestRunner_B.pass
    Running --> Fail: TestRunner_B.fail
    Running --> Eventually: all conjectures exhausted

    Fail --> CorrectUpstream: backward correction
    CorrectUpstream --> CONJ: ∆⁻(remove old) + ∆⁺(append new)

    Pass --> Done: VerdictOutputter
    Eventually --> Done: VerdictOutputter (unfixed)

    Done --> [*]: next bug
```

---

## Conjecture Escalation Order

```mermaid
flowchart LR
    subgraph ORDER["TestRunner_B tries conjectures in order"]
        C1["C1: light model + D-context + generic prompt + aggro compression"]
        C2["C2: light model + D-context + CoT prompt + aggro compression"]
        C3["C3: light model + E-context + generic prompt + aggro compression"]
        C4["C4: heavy model + E-context + CoT prompt + gentle compression"]
    end

    C1 -->|fail| C2
    C2 -->|fail| C3
    C3 -->|fail| C4
    C4 -->|fail| eventually
```

---

## Edge Label Summary

| Edge | Label | Why |
|---|---|---|
| `Checkout → Sources` | `im+` | New bug = new source files |
| `Sources → TestRunner_A` | `i` | Re-localization needs re-run |
| `TestRunner_A → FailLocator` | `im+` | New failures → new locations |
| `FailLocator → ContextCollector` | `im+` | New location → new context |
| `ContextCollector → PromptBuilder` | `im+` | New context → new prompt |
| `PromptBuilder → TokenOptimizer` | `im+` | New prompt → optimize |
| `TokenOptimizer → LLMCaller` | `im+` | Optimize → API call |
| `LLMCaller → PatchApplier` | `im+` | New patch → apply |
| `PatchApplier → TestRunner_B` | `im+` | Applied → validate |
| `TestRunner_B.pass → VerdictOut` | `im+` | Passed → emit |
| `TestRunner_B.eventually → VerdictOut` | `im+` | Exhausted → emit (unfixed) |
| `← TestRunner_B → ContextCollector` | **backward** | `∆⁻`(D) + `∆⁺`(E) — escalate context |
| `← TestRunner_B → PromptBuilder` | **backward** | `∆⁻`(generic) + `∆⁺`(CoT) — change strategy |
| `← TestRunner_B → LLMCaller` | **backward** | `∆⁻`(light) + `∆⁺`(heavy) — upgrade model |
| `← TestRunner_B → TokenOptimizer` | **backward** | `∆⁻`(aggro) + `∆⁺`(gentle) — reduce compression |
