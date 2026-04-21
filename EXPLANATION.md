# Detailed Explanation of the Mermaid System Pipeline

This document explains the Mermaid sequence diagram for the AutoML-Agent system in detail.

## How to read the Mermaid diagram

- **Participants** are system roles/components (`User`, `PromptAgent`, `AgentManager`, etc.).
- **Arrows (`->>`)** show messages or calls.
- **`par ... and ... end`** means concurrent branches (data and model flows run in parallel per plan).
- **`loop`** means iterative retries.
- **`alt / else`** means conditional branch based on success/failure.

---

## 1) User request enters the system

1. The **User** sends a natural-language ML request.
2. **PromptAgent** converts that free text into structured JSON requirements via `parse()` (or optional `parse_openai()` path).
3. That JSON is handed to **AgentManager** as the formal input for orchestration.

Why this matters:
- Everything downstream assumes structured fields (`problem`, `dataset`, `model`, etc.), so prompt parsing is the normalization gate.

---

## 2) AgentManager INIT and PLAN stages

4. In `INIT`, Manager validates request relevance (`_is_relevant`) and adequacy (`_is_enough`), then creates a short requirement summary.
5. In `PLAN`, Manager calls `make_plans()` to generate multiple candidate end-to-end plans (`n_plans`).

Why this matters:
- The framework intentionally creates **multiple candidate plans** to improve robustness and diversity before execution.

---

## 3) Retrieval-Augmented Planning (RAP) knowledge fusion

6. During planning, Manager may call `retrieve_knowledge(...)` (if RAP enabled).
7. That retriever composes summaries from:
   - web search,
   - arXiv,
   - PapersWithCode,
   - Kaggle notebooks,
   and fuses them into one planning-knowledge chunk used to guide plan generation.

Why this matters:
- This is the “knowledge grounding” stage that aligns with the paper’s RAG-like strategy for up-to-date planning.

---

## 4) Parallel per-plan execution: Data lane + Model lane

8. In `ACT`, Manager executes each plan by calling `execute_plan(plan)`.
9. `execute_plan` invokes:
   - **DataAgent** for data manipulation strategy,
   - **ModelAgent** for modeling/optimization strategy.

### Data branch
10. DataAgent optionally decomposes the plan (`understand_plan`) if enabled.
11. DataAgent calls `retrieve_datasets(...)` to discover dataset sources (user-upload/link/direct search/hub/fallback).
12. DataAgent returns a detailed data-side execution narrative (not code).

### Model branch
13. ModelAgent optionally decomposes the plan.
14. ModelAgent calls `retrieve_models(...)`, prioritized across HuggingFace → Kaggle → PyTorch ecosystem by modality.
15. ModelAgent returns top-k-focused modeling/optimization narrative (not code).

Why the Mermaid `par` block is correct:
- Manager uses multiprocessing pool for plan execution and verification, so data/model work is conceptually parallelized per plan and across plans.

---

## 5) PRE_EXEC verification (before any code is generated)

16. Manager runs `verify_solution(solution)` against each plan’s Data+Model outputs.
17. Each candidate is marked pass/fail; only passed solutions move forward.

Why this matters:
- This is a **pre-code quality gate**, reducing wasted coding attempts.

---

## 6) EXEC stage: turn selected strategy into concrete code

18. Manager combines successful data/model outputs into one “best” implementation instruction.
19. Manager calls `implement_solution(...)`, which:
   - loads task template from `prompt_pool/{task}.py`,
   - creates `OperationAgent`,
   - delegates code synthesis/execution.

Why Mermaid shows template loading:
- Operation starts from a task skeleton (e.g., tabular template), then fills in details.

---

## 7) OperationAgent iterative code loop (`loop` block)

20. OperationAgent builds a coding prompt from:
   - instructions,
   - previous code,
   - previous error logs.
21. It extracts generated Python code, writes to `./agent_workspace/exp/...py`.
22. It validates by running the script (`execute_script`), capturing return code + logs.
23. If failed, it retries up to `n_attempts`.

Why Mermaid `loop` is correct:
- The exact behavior is iterative self-fix with error-informed retries.

---

## 8) POST_EXEC verification and terminal branch (`alt/else`)

24. If execution succeeded (`rcode == 0`), Manager does post-execution requirement verification (Pass/Fail).
25. **Pass** → end and return final code/pipeline.
26. **Fail** → Manager revises instructions (`REV`) and re-enters `EXEC` for another coding round.

Why Mermaid `alt/else` is correct:
- There are explicit success/failure branches in state transitions.

---

## 9) Why this sequence design is important

- It separates concerns:
  - Prompt parsing,
  - planning,
  - data specialization,
  - model specialization,
  - code synthesis/execution,
  - verification gates.
- It has **two verification checkpoints**:
  1) before coding (`PRE_EXEC`),
  2) after coding (`POST_EXEC`),
  which improves reliability compared to one-shot code generation.
