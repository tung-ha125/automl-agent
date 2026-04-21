# AutoML-Agent: Detailed End-to-End Explanation (Step-by-Step)

This document explains how the project works from user request to final code, with each step specifying:

- **Input**
- **Output format**
- **Prompt used in that step**

It is implementation-grounded and aligned with the repository structure.

---

## 0) System Overview

AutoML-Agent is a multi-agent framework for full-pipeline ML automation.  
Main runtime components:

- **PromptAgent** ([`prompt_agent/__init__.py`](./prompt_agent/__init__.py))
- **AgentManager** ([`agent_manager/__init__.py`](./agent_manager/__init__.py))
- **DataAgent** ([`data_agent/__init__.py`](./data_agent/__init__.py))
- **ModelAgent** ([`model_agent/__init__.py`](./model_agent/__init__.py))
- **OperationAgent** ([`operation_agent/__init__.py`](./operation_agent/__init__.py))
- Retrieval modules:
  - [`agent_manager/retriever.py`](./agent_manager/retriever.py)
  - [`data_agent/retriever.py`](./data_agent/retriever.py)
  - [`model_agent/retriever.py`](./model_agent/retriever.py)
- Task templates in [`prompt_pool/`](./prompt_pool/)

---

## 1) User Request Parsing

### Purpose
Convert free-form user intent into structured machine-readable requirements.

### Input
- `instruction: str` (natural-language user request)

### Output format
- **JSON object** (or JSON string) describing requirements such as:
  - problem/task type
  - dataset/source hints
  - model constraints/preferences
  - evaluation targets
  - deployment/reporting expectations

### Prompt used
Used by `PromptAgent.parse(...)` / `PromptAgent.parse_openai(...)`:
- **System intent**: act as a parser that maps instruction → schema-valid JSON.
````
You are an assistant project manager in the AutoML development team. 
Your task is to parse the user's requirement into a valid JSON format using the JSON specification schema as your reference. Your response must exactly follow the given JSON schema and be based only on the user's instruction. 
Make sure that your answer contains only the JSON response without any comment or explanation because it can cause parsing errors.

#JSON SPECIFICATION SCHEMA#
```json
{json_specification}
```

Your response must begin with "```json" or "{{" and end with "```" or "}}", respectively.
````
`json_specification` was defined in [`prompt_agent/WizardLAMP/template_schema.json`](./prompt_agent/WizardLAMP/template_schema.json)

- **User template intent**: “Carefully parse #Instruction# into JSON and return only JSON.”
```
Please carefully parse the following #Instruction#. 
Your response can only begin with "```json" or "{{" and end with "```" or "}}" without saying any word or explain.

#Instruction#
{instruction}

#Valid JSON Response#
```

Implementation references:
- [`prompt_agent/__init__.py`](./prompt_agent/__init__.py)
- [`prompt_agent/schema.json`](./prompt_agent/schema.json)

---

## 2) Relevance Check (INIT sub-step)

### Purpose
Reject non-ML requests early.

### Input
- Raw user prompt text.

### Output format
- **Boolean-like decision** (Yes/No), interpreted as relevant or not relevant.

### Prompt used
- “Is this statement relevant to machine learning or artificial intelligence? Answer only Yes or No.”

Implementation reference:
- [`AgentManager._is_relevant` in `agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 3) Requirement Sufficiency Check (INIT sub-step)

### Purpose
Determine whether the request includes enough information to proceed (especially problem + data context).

### Input
- Raw prompt and/or parsed requirements.

### Output format
- Short structured judgment:
  - `yes/no`
  - optional reason

### Prompt used
- Prompt asks whether key required project fields are present and requests concise yes/no-style decision with reason.

Implementation reference:
- [`AgentManager._is_enough` in `agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 4) Requirement Summarization (INIT sub-step)

### Purpose
Build a concise requirement summary used downstream in planning and retrieval.

### Input
- Parsed requirement JSON.

### Output format
- A compact plain-text summary string (`req_summary`).

### Prompt used
```
Please briefly summarize the user's request represented in the following JSON object into a single paragraph based on how you understand it.
{self.user_requirements}
```

Implementation reference:
- [`agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 5) Retrieval-Augmented Planning Knowledge (PLAN sub-step)

### Purpose
Gather external/project-side knowledge to ground planning.

### Input
- Requirement summary
- task/domain/area fields from parsed requirements

### Output format
- Source-specific summaries + one fused planning knowledge block.

### Prompt used (by source)
1. **Web retrieval summary prompt**  
   - generate query, retrieve snippets/docs, summarize relevance.
2. **arXiv summary prompt**  
   - summarize methods, assumptions, and practical takeaways.
3. **PapersWithCode summary prompt**  
   - summarize benchmark/model insights.
4. **Kaggle summary prompt**  
   - summarize practical notebook strategies.
5. **Fusion prompt**  
   - merge all source summaries into actionable planning guidance.

Implementation references:
- [`agent_manager/retriever.py`](./agent_manager/retriever.py)
- [`retrieve_knowledge(...)` call path in `agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 6) Multi-Plan Generation (PLAN)

### Purpose
Generate multiple candidate end-to-end AutoML plans.

### Input
- Parsed requirements
- Requirement summary
- Fused retrieval knowledge (if enabled)
- plan constraints

### Output format
- `List[str]` plans (`n_plans`, typically >1), each describing a full pipeline strategy.

### Prompt used
- Project-manager-style planning prompt requiring:
  - complete lifecycle coverage
  - actionable steps
  - consistency with user constraints
  - practical implementation focus

Implementation reference:
- [`AgentManager.make_plans(...)` in `agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 7) Per-Plan Parallel Execution: Data + Model Lanes (ACT)

For each candidate plan, `AgentManager.execute_plan(plan)` runs data and model reasoning in a parallel-style workflow.

### 7A) DataAgent plan understanding + dataset retrieval

#### Purpose
Produce data-side execution strategy (source, preprocessing, augmentation, checks).

#### Input
- `plan: str`
- `user_requirements`
- optional uploaded/local `data_path`

#### Output format
- Detailed **textual data execution result** including:
  - dataset acquisition/selection strategy
  - preprocessing and quality steps
  - expected intermediate outcomes

#### Prompt used
1. **Plan decomposition prompt** (`understand_plan`)  
   - distill the plan into reproducible data tasks.
2. **Execution synthesis prompt**  
   - generate practical data operations and expected outcomes.

Implementation references:
- [`data_agent/__init__.py`](./data_agent/__init__.py)
- [`data_agent/retriever.py`](./data_agent/retriever.py)

---

### 7B) ModelAgent plan understanding + model retrieval

#### Purpose
Produce model-side strategy with candidate models and optimization ideas.

#### Input
- `project_plan`
- `data_result` from DataAgent
- `user_requirements`
- `k` candidate target count

#### Output format
- Detailed **textual model execution result** including:
  - candidate models (top-k style)
  - training/optimization guidance
  - metric-aware evaluation expectations

#### Prompt used
1. **Plan decomposition prompt** (`understand_plan`)  
   - convert plan to model-centric tasks.
2. **Execution synthesis prompt**  
   - produce concrete modeling and HPO instructions with expected outcomes.

Implementation references:
- [`model_agent/__init__.py`](./model_agent/__init__.py)
- [`model_agent/retriever.py`](./model_agent/retriever.py)

---

## 8) Pre-Execution Verification (PRE_EXEC)

### Purpose
Filter weak/invalid plan outputs before code generation.

### Input
- Combined per-plan result:
  - `solution["data"]`
  - `solution["model"]`
  - `user_requirements`

### Output format
- Pass/fail label per candidate solution (boolean-like).

### Prompt used
- “Given proposed solution and user requirements, answer only Pass or Fail.”

Implementation reference:
- [`AgentManager.verify_solution(...)` in `agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 9) Instruction Synthesis for Coding (EXEC preparation)

### Purpose
Turn selected validated strategy into implementable coding instructions.

### Input
- Passing data/model narratives
- requirement summary
- selected task type

### Output format
- A single comprehensive instruction block for code generation.

### Prompt used
- Instruction-composition prompt asks model to act like a senior MLOps engineer and produce complete implementation guidance from selected strategy.

Implementation reference:
- [`agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 10) Template Injection (EXEC preparation)

### Purpose
Seed code generation with task-specific scaffold.

### Input
- Task name (e.g., tabular classification, text classification, forecasting, etc.)

### Output format
- Template code/context loaded from [`prompt_pool/{task}.py`](./prompt_pool/).

### Prompt used
- No separate user-facing prompt; template becomes part of the coding context passed to OperationAgent.

Implementation references:
- [`prompt_pool/`](./prompt_pool/)
- [`agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 11) Iterative Code Generation & Execution Loop (EXEC)

### Purpose
Generate runnable Python code, execute it, and self-correct using logs.

### Input (per attempt)
- coding instructions
- previous code
- previous execution logs/errors
- template context
- downstream task metadata

### Output format
- `implementation_result` dictionary, typically containing:
  - `rcode` (int)
  - `action_result` (execution output summary)
  - `code` (generated Python source)
  - `error_logs` (aggregated failures)

### Prompt used
OperationAgent coding prompt includes:
- current objective
- prior failed code/logs (if any)
- requirement to return executable code
- iteration-aware repair behavior

Implementation references:
- [`operation_agent/__init__.py`](./operation_agent/__init__.py)
- [`operation_agent/execution.py`](./operation_agent/execution.py)

---

## 12) Post-Execution Verification (POST_EXEC)

### Purpose
Check whether generated code + observed result satisfy user requirements.

### Input
- final/generated code
- execution logs/results
- original requirements

### Output format
- Pass/fail decision.

### Prompt used
- Verification prompt asking if produced implementation fulfills requirements, with strict Pass/Fail output.

Implementation reference:
- [`agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 13) Revision Cycle (REV) on Failure

### Purpose
If POST_EXEC fails, revise instructions and retry implementation.

### Input
- previous instructions
- generated code
- error logs
- verification feedback

### Output format
- revised instruction text fed back to EXEC loop.

### Prompt used
- Revision prompt focused on correcting concrete issues revealed in logs/verifier feedback while preserving valid parts.

Implementation reference:
- [`agent_manager/__init__.py`](./agent_manager/__init__.py)

---

## 14) End State (END)

### Purpose
Return final accepted solution artifact.

### Input
- successful implementation result and pass decision.

### Output format
- Final code artifact + run/validation outcome summary.

### Prompt used
- No major new generation prompt; this is a terminal orchestration state after successful verification.

---

## Compact Pipeline Table

| Step | Input | Output format | Prompt type |
|---|---|---|---|
| Parse request | user instruction | JSON requirements | schema-constrained parser prompt |
| Relevance check | raw prompt | Yes/No | ML relevance classifier prompt |
| Sufficiency check | prompt/requirements | yes/no + reason | requirement completeness prompt |
| Retrieval | req summary + task/domain | source summaries + fused knowledge | source summarization + fusion prompts |
| Plan generation | requirements + fused knowledge | list of plans | project-manager planning prompt |
| Data execution | plan + req + data_path | data strategy text | decomposition + execution prompts |
| Model execution | plan + data result + req | model strategy text | decomposition + execution prompts |
| PRE_EXEC verify | data/model outputs + req | Pass/Fail | solution-vs-requirements verifier prompt |
| Instruction synthesis | selected passing solution | coding instruction text | MLOps implementation prompt |
| Code loop | instructions + code/log history | `implementation_result` dict | iterative coding/repair prompt |
| POST_EXEC verify | code + logs + req | Pass/Fail | final requirement verifier prompt |
| REV | fail evidence + prior instruction | revised instruction | failure-driven revision prompt |

---

## Quick Access: Key Files

- Orchestration: [`agent_manager/__init__.py`](./agent_manager/__init__.py)
- Planning retrieval: [`agent_manager/retriever.py`](./agent_manager/retriever.py)
- Prompt parsing: [`prompt_agent/__init__.py`](./prompt_agent/__init__.py)
- Data strategy:
  - [`data_agent/__init__.py`](./data_agent/__init__.py)
  - [`data_agent/retriever.py`](./data_agent/retriever.py)
- Model strategy:
  - [`model_agent/__init__.py`](./model_agent/__init__.py)
  - [`model_agent/retriever.py`](./model_agent/retriever.py)
- Code generation/execution:
  - [`operation_agent/__init__.py`](./operation_agent/__init__.py)
  - [`operation_agent/execution.py`](./operation_agent/execution.py)
- Task templates: [`prompt_pool/`](./prompt_pool/)
- Configuration: [`configs.py`](./configs.py)
- Project overview: [`README.md`](./README.md)
- Existing explanation docs:
  - [`EXPLANATION.md`](./EXPLANATION.md)
  - [`DETAILED_EXPLANATION.md`](./DETAILED_EXPLANATION.md)

---

## Practical Notes

1. **Two verification gates** improve reliability:
   - PRE_EXEC (before code)
   - POST_EXEC (after code execution)

2. **Template-driven generation** reduces cold-start code errors.

3. **Iterative repair loop** uses runtime logs, not only static reasoning.

4. Final quality depends heavily on:
   - API/backend setup in [`configs.py`](./configs.py)
   - retrieval quality/availability
   - clarity/completeness of parsed requirements
