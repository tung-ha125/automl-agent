# AutoML-Agent: Detailed System Explanation

This document provides a practical, implementation-grounded walkthrough of the AutoML-Agent repository.
It explains each process stage with:

- Inputs
- Outputs
- Core functions/files
- Prompt templates used in code
- Operational notes (parallelism, retries, validation)

---

## 1) High-Level Purpose

AutoML-Agent is a multi-agent orchestration framework that turns a user’s natural-language ML request into:

1. Structured requirements
2. Multiple candidate plans
3. Data and model execution instructions
4. Runnable Python code
5. Validation outcomes (pre-code and post-code)

At runtime, the central orchestrator is `AgentManager`, which coordinates specialized agents:

- `PromptAgent`: requirement parsing
- `DataAgent`: data strategy
- `ModelAgent`: model strategy
- `OperationAgent`: code generation/execution

---

## 2) Core Runtime State Machine (AgentManager)

`AgentManager.initiate_chat(...)` runs a staged state machine:

1. `INIT`
2. `PLAN`
3. `ACT`
4. `PRE_EXEC`
5. `EXEC`
6. `POST_EXEC`
7. `REV` / `END`

### State-by-state summary

| State | Primary Goal | Main Input | Main Output |
|---|---|---|---|
| INIT | Verify request & parse into JSON | User prompt | `user_requirements` JSON + request summary |
| PLAN | Generate multiple candidate plans | Parsed requirements + retrieved knowledge | `self.plans` |
| ACT | Execute plans in parallel by data/model specialists | Candidate plans | `action_results` (data+model textual outputs) |
| PRE_EXEC | Verify proposed solutions before coding | `action_results` | pass/fail labels per candidate |
| EXEC | Convert selected solution to code and run | Passed results + template code | `implementation_result` (rcode, logs, code) |
| POST_EXEC | Verify generated code/run result against requirements | code + run logs + requirements | Pass to END or revise and retry |
| REV | Revise plan/instructions after failures | failure reasons/logs | updated plans/instructions |

---

## 3) End-to-End Process with Inputs/Outputs

## 3.1 User Prompt → Structured Requirements

### Component
- `PromptAgent` (`prompt_agent/__init__.py`)

### Inputs
- `instruction` (free-text user request)

### Core methods
- `parse(instruction, return_json=False)`
- `parse_openai(instruction, return_json=False)`

### Prompts used

System profile (schema-constrained parser role):
- The agent is instructed to convert user instructions into valid JSON aligned to `template_schema.json`.

User prompt skeleton:
- “Please carefully parse the following #Instruction# ... response can only begin with ` ```json ` or `{` ...”

### Outputs
- JSON requirement object with project fields (problem, dataset, model, etc.)

### Notes
- `parse_openai` uses OpenAI response format `json_object`.
- `parse` uses the `prompt-llm` backend (vLLM-style endpoint configured in `configs.py`).

---

## 3.2 Requirement Relevance & Sufficiency Checks

### Component
- `AgentManager` (`agent_manager/__init__.py`)

### Inputs
- Raw user prompt
- Parsed requirement JSON

### Core methods
- `_is_relevant(msg)`
- `_is_enough(msg)`

### Prompts used

For relevance:
- “Is the following statement relevant to machine learning or artificial intelligence? ... Answer only 'Yes' or 'No'”

For sufficiency:
- Ask whether requirements contain essential project info (especially problem + dataset), returning `yes/no;reason`.

### Outputs
- Relevance boolean
- Sufficiency boolean + reason
- Requirement summary paragraph (`self.req_summary`)

---

## 3.3 Retrieval-Augmented Planning (RAP)

### Component
- Manager retriever (`agent_manager/retriever.py`)

### Inputs
- `user_requirements`
- `user_requirement_summary`
- selected LLM client/model

### Sub-processes and their I/O

#### A) Web Search Retrieval
- Function: `retrieve_websearch(...)`
- Input: requirement summary
- Process:
  1. Generates short search query via LLM
  2. Calls `search_web(...)` utility
  3. Loads HTML/PDF snippets
  4. Retrieves relevant chunks via BM25
  5. Summarizes via LLM
- Output: one web knowledge summary paragraph

#### B) arXiv Retrieval
- Function: `retrieve_arxiv(...)`
- Input: downstream task + application domain
- Process:
  1. Build arXiv query
  2. Load top papers, parse PDFs
  3. Chunk/retrieve relevant passages
  4. Summarize via LLM
- Output: one arXiv summary paragraph

#### C) PapersWithCode Retrieval
- Function: `retrieve_paperswithcode(...)`
- Input: downstream task + area
- Process:
  1. Filter local PWC datasets/evaluation tables JSON
  2. Create candidate docs
  3. BM25 retrieve relevant content
  4. Summarize via LLM
- Output: one PWC summary paragraph

#### D) Kaggle Notebook Retrieval
- Function: `retrieve_kaggle(...)`
- Input: downstream task + application domain
- Process:
  1. Query Kaggle notebooks
  2. Pull notebook content
  3. Retrieve relevant chunks
  4. Summarize key insights (not raw code)
- Output: one Kaggle summary paragraph

#### E) Final RAP Fusion
- Function: `retrieve_knowledge(...)`
- Input: all source summaries
- Process: aggregate summaries and produce final planning guidance list
- Output: consolidated planning knowledge used in `make_plans(...)`

---

## 3.4 Multi-Plan Generation

### Component
- `AgentManager.make_plans(...)`

### Inputs
- Parsed requirements (`self.user_requirements`)
- RAP knowledge (`self.plan_knowledge`)
- Planning constraints (`plan_conditions`)

### Prompt used
- A project-manager role prompt requiring complete end-to-end actionable plans.
- Constraints include being up-to-date, self-contained, and covering full ML lifecycle.

### Outputs
- `self.plans`: list of plan texts (`n_plans`, default 3)

### Notes
- Each plan is generated independently to preserve diversity.

---

## 3.5 DataAgent Execution (Per Plan)

### Component
- `DataAgent` (`data_agent/__init__.py`)
- `data_agent/retriever.py`

### Inputs
- `plan`
- `user_requirements`
- `data_path` (if user-uploaded files exist)

### Steps and prompts

1. Optional decomposition (`understand_plan`) prompt asks the data agent to summarize plan into reproducible data tasks:
   - retrieve/collect data
   - preprocess
   - augment
   - characterize data

2. Source retrieval via `retrieve_datasets(...)` with priority/fallback logic:
   - user-upload/user-link
   - direct search over HuggingFace/Kaggle/PyTorch/TFDS/UCI/OpenML
   - infer-search fallback using web + LLM

3. Execution explanation prompt asks for detailed data operations and expected outcomes (without code).

### Outputs
- One detailed text artifact: data pipeline instructions + expected outcomes

---

## 3.6 ModelAgent Execution (Per Plan)

### Component
- `ModelAgent` (`model_agent/__init__.py`)
- `model_agent/retriever.py`

### Inputs
- `project_plan`
- `data_result` from DataAgent
- `k` (`n_candidates`, default 3)

### Steps and prompts

1. Optional decomposition (`understand_plan`) into model-specific actions.
2. Candidate retrieval via `retrieve_models(...)`:
   - HuggingFace first
   - Kaggle models next
   - PyTorch ecosystem fallback by modality (vision/text/audio/graph)
3. Execution prompt asks for detailed top-k modeling + hyperparameter optimization instructions and expected metrics/complexity.

### Outputs
- One detailed text artifact: modeling/optimization instructions + expected top-k candidates

---

## 3.7 Pre-Code Verification

### Component
- `AgentManager.verify_solution(...)`

### Inputs
- Combined per-plan outputs:
  - `solution["data"]`
  - `solution["model"]`
  - `user_requirements`

### Prompt used
- “Given proposed solution and user requirements, answer only Pass or Fail.”

### Outputs
- Boolean pass/fail per plan
- Manager keeps only passing candidates for implementation stage

---

## 3.8 Code Instruction Synthesis + OperationAgent Coding Loop

### Components
- `AgentManager` (instruction synthesis)
- `OperationAgent` (`operation_agent/__init__.py`)
- script execution utility (`operation_agent/execution.py`)

### Inputs
- Passed data/model narratives
- Request summary
- Task template from `prompt_pool/{task}.py`

### Instruction synthesis prompt (Manager)
- Select one promising solution from passed candidates.
- Produce complete coding guidance for MLOps engineer.
- Emphasize dataset/model paths and implementation completeness.

### OperationAgent coding prompt
Each iteration includes:
- downstream task
- current instructions
- previously written code
- prior error logs
- target pipeline scope (full pipeline vs modeling-only)

### Execution loop behavior
1. Generate code block from LLM response.
2. Write code to `./agent_workspace/exp/{code_path}.py`.
3. Run code via `execute_script(...)` (`CUDA_VISIBLE_DEVICES=... python -u ...`).
4. If error: append logs and retry up to `n_attempts`.

### Outputs
- `implementation_result` dictionary:
  - `rcode`
  - `action_result` (stdout/stderr summary)
  - `code`
  - `error_logs`

---

## 3.9 Post-Code Verification and Revision

### Component
- `AgentManager` (`POST_EXEC` + `REV` behavior)

### Inputs
- Generated code
- Execution logs
- User requirements

### Prompt used
- “Verify whether code and results satisfy requirements; answer Pass/Fail.”

### Outputs
- If Pass: `END` with final solution code
- If Fail and revision budget remains:
  - Manager revises coding instruction using previous instruction + code + error logs
  - Returns to `EXEC`

---

## 4) Prompt Inventory (Quick Reference)

Below are the major prompt classes used by the system:

1. Parsing prompt (PromptAgent): convert free text to JSON schema
2. Relevance prompt (Manager): ML/AI relevance check
3. Sufficiency prompt (Manager): essential information check
4. Planning prompt (Manager): generate complete actionable plans
5. Retrieval summary prompts (Retriever): summarize source-specific evidence
6. Data decomposition + execution prompts (DataAgent)
7. Model decomposition + execution prompts (ModelAgent)
8. Pre-exec verification prompt (Manager): pass/fail for textual solutions
9. Code-instruction synthesis prompt (Manager)
10. Coding prompt (OperationAgent): write/fix runnable code
11. Post-exec verification prompt (Manager)
12. Revision prompt (Manager): improve coding guidance from failure evidence

---

## 5) Inputs/Outputs by Module (Compact Table)

| Module | Main Inputs | Main Outputs |
|---|---|---|
| PromptAgent | user text | requirement JSON |
| AgentManager | requirement JSON + summaries + agent outputs | plans, selected instructions, final code/result |
| Manager Retriever | requirement summary + task/domain/area fields | fused knowledge chunk |
| DataAgent | plan + requirements + data path | data execution narrative |
| ModelAgent | plan + data narrative + requirements | model execution narrative |
| OperationAgent | coding instructions + template + errors | runnable code + run logs + status |

---

## 6) Parallelism, Retries, and Robustness Mechanisms

### Parallelism
- Plan executions are parallelized using multiprocessing pools.
- Pre-execution verification is also parallelized.

### Retry behavior
- LLM calls in many components are wrapped with retry loops.
- OperationAgent has iterative self-correction using execution logs.

### Validation checkpoints
- Checkpoint 1: `PRE_EXEC` verifies strategy quality before coding.
- Checkpoint 2: `POST_EXEC` verifies real code + results against requirements.

---

## 7) Configuration and Dependency Expectations

### LLM and API configuration
- Defined in `configs.py`:
  - `AVAILABLE_LLMs`
  - API key placeholders
  - task metrics map

### Utility dependencies
- Web search through SerpAPI wrapper (`search_web`) in `utils/__init__.py`.
- Kaggle API auth via `get_kaggle()`.

### Practical setup implications
- Production use requires valid API keys/endpoints.
- Some retrieval paths depend on optional external packages and service credentials.

---

## 8) Template-Driven Code Generation

Task templates in `prompt_pool/*.py` provide a scaffold for data preprocessing, training, evaluation, deployment, and reporting.

Example (`tabular_classification.py`):
- placeholders for data loading, splitting, preprocessing, model definition, training, evaluation, deployment
- expected result prints include performance and complexity summaries

This template is fed into OperationAgent as the initial code context and then iteratively refined.

---

## 9) Practical Reading Guide for New Contributors

If you are new to the repository, read in this order:

1. `README.md` (project objective + basic usage)
2. `agent_manager/__init__.py` (orchestration state machine)
3. `prompt_agent/__init__.py` (request parsing)
4. `agent_manager/retriever.py` (RAP knowledge flow)
5. `data_agent/*` then `model_agent/*` (specialist execution)
6. `operation_agent/*` (code synthesis + execution loop)
7. `prompt_pool/*.py` (task scaffolds)

---

## 10) Key Takeaways

- The system is not just a single LLM prompt; it is a staged multi-agent pipeline with explicit orchestration.
- It separates strategic reasoning (plans/instructions) from executable implementation (code loop).
- Reliability is improved through two validation gates and iterative error-driven correction.
- The quality of retrieval and configuration strongly affects planning and downstream implementation quality.
