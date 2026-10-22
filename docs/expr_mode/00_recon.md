# Phase 0 recon: stock `fin_factor` on latest `main`

Recon and documentation only. This change does not add expression-factor behavior and does not edit CoSTEER, experiment generation, or workflow internals.

Official lineage: this checkout is `ckosun/RD-Agent`, parent `microsoft/RD-Agent`. `HEAD` matches `microsoft/RD-Agent` `main` and the GitHub tag `v1.0.0`. No third-party RD-Agent fork was used. AlphaAgent notes below are ideas from `https://github.com/RndmVariableQ/AlphaAgent` only. No AlphaAgent code is copied.

## 1. Pin

| Item | Value |
| --- | --- |
| Fork | `ckosun/RD-Agent` (`parent`: `microsoft/RD-Agent`, default branch `main`) |
| Branch this doc was written from | `main` at the time of branching; work branch is `expr-mode` |
| Commit | `484776c211e4fbbeef03e0ec00d6bbee7362a4f4` |
| Subject | `chore(main): release 1.0.0 (#1286)` |
| Committer date | 2026-09-23 19:05:10 +0800 (`you-n-g`) |
| Upstream tag on that SHA | `v1.0.0` (GitHub tag object type `commit`, same SHA). `v0.8.0` is an ancestor: `274e274d5dbb72cc2ea139d1a7c93d73ce9b1198` (changelog date 2025-11-03) |
| Local tags | None. `git describe --tags --always` prints `484776c`. The tag exists on GitHub and was not present in this clone |
| Declared package version | `pyproject.toml` version is dynamic via setuptools-scm (`version_scheme = guess-next-dev`). Changelog heading is `1.0.0`. `.bumpversion.cfg` still says `current_version = 0.0.0` |
| Installed dist version in the smoke venv | `0.1.dev1023` (`importlib.metadata.version("rdagent")`). That string is setuptools-scm on a tagless clone (`git rev-list --count HEAD` is 1023), not a second release |

`requires-python` is `>=3.10`. Classifiers list 3.10 and 3.11. Constraint files exist only for `constraints/3.10.txt` and `constraints/3.11.txt`.

## 2. Non-Docker install smoke of `rdagent fin_factor`

Host observed on 2026-10-22:

- Python `3.12.3` (`/usr/bin/python3`). No `python3.10` or `python3.11` binary.
- `conda` not installed. `docker` not installed.
- No `OPENAI_API_KEY`, `CHAT_MODEL`, or Azure key in the environment.
- `python3-venv` was missing until `python3.12-venv` was installed so a venv could be created. That is an environment package, not a repo change.

`make install` would select `constraints/$(python -V major.minor).txt`. On this host that path is `constraints/3.12.txt`, which is not in the tree. The smoke therefore did not use `make install`. It installed the package editable, without a constraints file:

```text
python3 -m venv /tmp/rdagent-venv
/tmp/rdagent-venv/bin/python -m pip install -U pip setuptools wheel
/tmp/rdagent-venv/bin/python -m pip install -e /workspace
```

That install completed. `rdagent fin_factor --help` exited 0. Typer options are `--path`, `--step-n`, `--loop-n`, `--all-duration`, `--checkout` / `--no-checkout`. Import prints a PyMuPDF `fitz` deprecation warning from the installed `pymupdf` stack.

### Run 1: `rdagent fin_factor --loop-n 0`

Working directory `/tmp/fin-factor-smoke`. Exit code 1. The loop body never started. Construction fails inside `FactorRDLoop.__init__` → `QlibFactorScenario.__init__` → `get_runtime_environment` → `get_factor_env`.

`get_factor_env` (`rdagent/components/coder/factor_coder/config.py`) builds:

```text
LocalEnv(conf=CondaConf(conda_env_name=os.environ.get("CONDA_DEFAULT_ENV")))
```

`CONDA_DEFAULT_ENV` was unset, so `conda_env_name` was `None`. Pydantic raised:

```text
ValidationError: 1 validation error for CondaConf
conda_env_name
  Input should be a valid string [type=string_type, input_value=None, input_type=NoneType]
```

Trace stops at `rdagent/app/cli.py` `fin_factor_cli` → `rdagent/app/qlib_rd_loop/factor.py` `main` → `FactorRDLoop(FACTOR_PROP_SETTING)` → `rdagent/components/workflow/rd_loop.py` `RDLoop.__init__` → `QlibFactorScenario`.

This run did not reach an LLM call, CoSTEER, qlib `qrun`, or feedback. API keys were not the first failure.

### Run 2: same command with `CONDA_DEFAULT_ENV=rdagent4qlib`

Working directory `/tmp/fin-factor-smoke2`. Exit code 1. `CondaConf._update_bin_path` shells out to `conda run -n rdagent4qlib ...`. `conda` is absent, so `bin_path` stays empty. `LocalEnv` then runs `python runtime_info.py` with `PATH` equal to `:/bin/:/usr/bin/:` and logs:

```text
timeout: failed to run command ‘python’: No such file or directory
```

`get_runtime_environment_by_env` (`rdagent/scenarios/shared/get_runtime_info.py`) searches stdout for a JSON object and raises:

```text
AttributeError: 'NoneType' object has no attribute 'group'
```

During that process the LiteLLM backend logger initialized with empty `openai_api_key` and the code default `chat_model='gpt-4-turbo'` (`rdagent/oai/llm_conf.py`, `LLMSettings.chat_model`). No successful chat completion happened.

### What a later run would still need

These were not reached, and they are still required for a real loop:

1. A `python` on `PATH` inside the factor `LocalEnv` (today that path is filled from `conda run` against `CONDA_DEFAULT_ENV`).
2. Price data. If `FACTOR_CoSTEER_DATA_FOLDER` / `FACTOR_CoSTEER_DATA_FOLDER_DEBUG` are missing, `get_data_folder_intro` calls `generate_data_folder_from_qlib`, which always uses `QTDockerEnv` (`rdagent/scenarios/qlib/experiment/utils.py`). That path needs Docker even when the factor coder itself is `LocalEnv`.
3. Backtest env. `QlibFBWorkspace.execute` follows `MODEL_CoSTEER_ENV_TYPE`, default `"conda"` (`rdagent/components/coder/model_coder/conf.py`). `QlibCondaEnv.prepare` expects `conda` and creates env `rdagent4qlib` (Python 3.10) with qlib pin `microsoft/qlib@2fb9380b342556ddb50a4b24e4fe8655d548b2b8` (`rdagent/utils/env.py`, `QlibCondaConf` / `QlibCondaEnv`). `make init-qlib-env` is a different recipe: conda env name `qlibRDAgent`, Python 3.8, `pip install pyqlib`.
4. LLM credentials. Defaults are `CHAT_MODEL` / `chat_model` `gpt-4-turbo` and `EMBEDDING_MODEL` `text-embedding-3-small`, backend `rdagent.oai.backend.LiteLLMAPIBackend`. `.env.example` shows the keys. None were set here.
5. Qlib cn data under `~/.qlib/qlib_data/cn_data` (`conf_*.yaml` `provider_uri`, and `factor_data_template/generate.py`).

## 3. Call chain

Verified class attribute, imported in the smoke venv:

```text
FactorRDLoop.steps == ['direct_exp_gen', 'coding', 'running', 'feedback', 'record']
```

`LoopMeta` collects public methods. Leading-underscore helpers (`_propose`, `_exp_gen`) are not steps. `direct_exp_gen` calls both.

### 3.1 Entry

| Hop | Path | Symbol | What it does |
| --- | --- | --- | --- |
| CLI | `rdagent/app/cli.py` | `fin_factor_cli` | Typer command `fin_factor`. Forwards `path`, `step_n`, `loop_n`, `all_duration`, `checkout` into `main`. Does not pass `base_features_path` or interaction queues |
| Script | `rdagent/app/qlib_rd_loop/factor.py` | `main` | `fire.Fire(main)` when run as a script. Builds or loads the loop, then `asyncio.run(factor_loop.run(...))` |
| UI server | `rdagent/log/server/app.py` | target `fin_factor` | Calls the same `main(**self.kwargs)`. `fin_factor` is interactive: the server injects `user_interaction_queues` unless the target is in `_TARGETS_WITHOUT_USER_INTERACTION` (`general_model`, `fin_factor_report` only) |

`main` (`factor.py`):

- `path is None` → `FactorRDLoop(FACTOR_PROP_SETTING)`.
- else → `FactorRDLoop.load(path, checkout=checkout)`.
- `checkout_path` overrides `checkout` with a `Path`.
- `_init_base_features(base_features_path)` then, only if `kwargs["user_interaction_queues"]` is set, `_set_interactor` and `_interact_init_params`.

`FactorRDLoop` subclasses `RDLoop`. It sets `skip_loop_error = (FactorEmptyError, CoderError)` and `skip_loop_error_stepname = "feedback"`. Its `running` calls `self.runner.develop(prev_out["coding"])` and raises `FactorEmptyError` when that returns `None`.

### 3.2 Settings object

`FACTOR_PROP_SETTING = FactorBasePropSetting()` in `rdagent/app/qlib_rd_loop/conf.py`.

Env prefix `QLIB_FACTOR_` (`SettingsConfigDict`). Defaults read from the installed object:

| Field | Default class path or value |
| --- | --- |
| `scen` | `rdagent.scenarios.qlib.experiment.factor_experiment.QlibFactorScenario` |
| `hypothesis_gen` | `rdagent.scenarios.qlib.proposal.factor_proposal.QlibFactorHypothesisGen` |
| `hypothesis2experiment` | `rdagent.scenarios.qlib.proposal.factor_proposal.QlibFactorHypothesis2Experiment` |
| `coder` | `rdagent.scenarios.qlib.developer.factor_coder.QlibFactorCoSTEER` |
| `runner` | `rdagent.scenarios.qlib.developer.factor_runner.QlibFactorRunner` |
| `summarizer` | `rdagent.scenarios.qlib.developer.feedback.QlibFactorExperiment2Feedback` |
| `evolving_n` | `10` |
| splits | train `2008-01-01`–`2014-12-31`, valid `2015-01-01`–`2016-12-31`, test `2017-01-01`–`2020-08-01` |

`evolving_n` is stored and logged via `PROP_SETTING.model_dump()`. A repo search shows it is not read by the loop or by CoSTEER. Coder iterations use `CoSTEERSettings.max_loop` (default 10, env prefix on the factor subclass `FACTOR_CoSTEER_`).

`BasePropSetting` also has `knowledge_base` and `knowledge_base_path`. Both stay `None`. `RDLoop.__init__` does not construct them. The knowledge base that factor coding actually uses is the CoSTEER one in section 4.3.

### 3.3 `RDLoop.__init__` wiring

`rdagent/components/workflow/rd_loop.py`, class `RDLoop(LoopBase, metaclass=LoopMeta)`:

1. `scen = import_class(PROP_SETTING.scen)()`.
2. `self.plan = {"features": ALPHA20, "feature_codes": {}}`. `ALPHA20` is a dict of qlib operator strings in `rdagent/utils/qlib.py` (for example `RESI5` → `Resi($close, 5)/$close`).
3. `self.hypothesis_gen = import_class(...)(scen)`.
4. `self.hypothesis2experiment = import_class(...)()` (no scenario argument).
5. `self.coder = import_class(...)(scen)`.
6. `self.runner = import_class(...)(scen)`.
7. `self.summarizer = import_class(...)(scen)`.
8. `self.trace = Trace(scen=scen)`.
9. `super().__init__()` builds the asyncio queue, `session_folder = LOG_SETTINGS.trace_path / "__session__"`, and a `WorkflowTracker`.

`_init_base_features` replaces `plan["features"]` when `base_features_path/base_factors.json` is a dict of expression strings that passes `validate_qlib_features`, and loads `*.py` into `plan["feature_codes"]`. The CLI does not expose `base_features_path`. The fire/`main` signature does.

### 3.4 Step `direct_exp_gen` (proposal + experiment generation)

Method: `RDLoop.direct_exp_gen`. It waits while `get_unfinished_loop_cnt(loop_idx) >= RD_AGENT_SETTINGS.get_max_parallel()`, then:

1. `_propose` → `hypothesis_gen.gen(self.trace, self.plan)`, then `_interact_hypo` if queues exist. Logs tag `hypothesis generation`.
2. `_exp_gen` → `hypothesis2experiment.convert(hypothesis, self.trace)`. Logs `exp.sub_tasks` under tag `experiment generation`.
3. Copies `plan["features"]` and `plan["feature_codes"]` onto `exp.base_features` / `exp.base_feature_codes`, and onto `exp.based_experiments[-1]` when that list is non-empty.

Return value stored on the loop: `{"propose": hypo, "exp_gen": exp}`.

#### Proposal classes and prompts

`QlibFactorHypothesisGen` (`rdagent/scenarios/qlib/proposal/factor_proposal.py`) subclasses `FactorHypothesisGen` (`rdagent/components/proposal/__init__.py`), which subclasses `LLMHypothesisGen`. `targets` is the string `"factors"`.

`LLMHypothesisGen.gen`:

- System prompt `rdagent/components/proposal/prompts.yaml` key `hypothesis_gen.system_prompt`. It embeds scenario text, `hypothesis_output_format`, `hypothesis_specification`, and optional `plan["user_instruction"]`.
- User prompt key `hypothesis_gen.user_prompt` (same file): history, last trial, SOTA trial, `RAG`.
- `APIBackend().build_messages_and_create_chat_completion(..., json_mode=True, json_target_type=dict[str, str])`.

`QlibFactorHypothesisGen.prepare_context` fills:

| Key | Source |
| --- | --- |
| `hypothesis_and_feedback` | `scenarios.qlib.prompts:hypothesis_and_feedback` when `trace.hist` is non-empty |
| `last_hypothesis_and_feedback` | `scenarios.qlib.prompts:last_hypothesis_and_feedback` |
| `RAG` | A short string. Under 15 history rows it says to try easy factors. At 15 or more it says to try high-IC factors |
| `hypothesis_output_format` | `scenarios.qlib.prompts:factor_hypothesis_output_format` |
| `hypothesis_specification` | `scenarios.qlib.prompts:factor_hypothesis_specification` |

Scenario text comes from `QlibFactorScenario.get_scenario_all_desc`. For hypothesis gen, `filtered_tag` is `"factors"`, and the method ignores that argument. The rendered scenario is background, source-data intro, interface, output format, and simulator. Prompts for those pieces live in `rdagent/scenarios/qlib/experiment/prompts.yaml` (`qlib_factor_background`, `qlib_factor_interface`, `qlib_factor_output_format`, `qlib_factor_simulator`, `qlib_factor_experiment_setting`). `get_source_data_desc` calls `get_data_folder_intro()`, which describes `daily_pv.h5` in the debug data folder.

Structured hypothesis JSON (`factor_hypothesis_output_format`):

```json
{
  "hypothesis": "...",
  "reason": "..."
}
```

`convert_response` does `json.loads` and builds `Hypothesis` (`rdagent/core/proposal.py`) with `hypothesis`, `reason`, and `.get` of `concise_reason`, `concise_observation`, `concise_justification`, `concise_knowledge`. The published schema does not contain the `concise_*` keys, so those attributes are `None` unless the model adds them. `Hypothesis.__str__` prints only `hypothesis` and `reason`. The same `concise_*` reads already exist in the `v0.8.0` copy of this file.

`factor_hypothesis_specification` tells the model to propose 1–5 factors, start simple, and return one hypothesis object even when it names several factors.

#### Experiment-generation classes and prompts

`QlibFactorHypothesis2Experiment.convert` is `FactorHypothesis2Experiment.convert` → `LLMHypothesis2Experiment.convert` (`@wait_retry(retry_n=5)`).

Prompts: `rdagent/components/proposal/prompts.yaml` keys `hypothesis2experiment.system_prompt` and `hypothesis2experiment.user_prompt`. JSON mode targets `dict[str, dict[str, str | dict]]`.

`prepare_context` passes `target_hypothesis`, `scenario` (`get_scenario_all_desc()`, or `action="factor"` when `trace.scen` is `QlibQuantScenario`), history filtered to factor actions, and `experiment_output_format` from `scenarios.qlib.prompts:factor_experiment_output_format`. `target_list` is `[]`. `RAG` is `None`.

Structured task JSON: an object whose keys are factor names. Each value has `description`, `formulation` (described as latex), and `variables` (name → description).

`convert_response` builds one `FactorTask` per key:

```text
FactorTask(factor_name, factor_description, factor_formulation, variables)
```

Then:

```text
exp = QlibFactorExperiment(tasks, hypothesis=hypothesis)
exp.based_experiments = [QlibFactorExperiment(sub_tasks=[])] + [
    t[0] for t in trace.hist if t[1] and isinstance(t[0], FactorExperiment)
]
```

It computes `unique_tasks` by skipping names already present on `based_experiments`, then assigns `exp.tasks = unique_tasks`.

`Experiment.__init__` (`rdagent/core/experiment.py`) stores the constructor sequence on `self.sub_tasks`. There is no `tasks` property. The duplicate-name filter writes an unused attribute. The coder and runner read `sub_tasks`. This assignment is already present at `v0.8.0`.

Because `based_experiments` always starts with one empty `QlibFactorExperiment`, a normal `fin_factor` proposal does not take the runner branch that executes `conf_baseline.yaml`.

#### Data objects at this hop

| Object | Actual name | Role |
| --- | --- | --- |
| Hypothesis | `rdagent.core.proposal.Hypothesis` | `hypothesis`, `reason`, four `concise_*` fields |
| Trace | `rdagent.core.proposal.Trace` | `hist: list[tuple[Experiment, ExperimentFeedback]]`, `dag_parent`, `idx2loop_id`, `current_selection` |
| Task | `FactorTask` in `rdagent/components/coder/factor_coder/factor.py` | Subclass of `CoSTEERTask`. Fields `factor_name`, `factor_formulation`, `variables`, `factor_resources`, `factor_implementation` (default `False`). Description is `Task.description` via the `factor_description` property. `version` defaults to 1 (`AbsTask`) |
| Experiment | `QlibFactorExperiment` | Subclass of `FactorExperiment`, which is the alias `FactorExperiment = Experiment` in the same `factor.py`. Workspace types are `QlibFBWorkspace` (experiment) and `FactorFBWorkspace` (each sub-task) |
| Base expressions | `QlibFactorExperiment.base_features: dict[str, str]` | Qlib operator strings. Filled from `RDLoop.plan`, initially `ALPHA20` |
| Base python | `base_feature_codes: dict[str, str]` | Optional `*.py` bodies from `base_features_path` |

`QlibFactorExperiment.__init__` sets `experiment_workspace` to a `QlibFBWorkspace` whose template folder is `rdagent/scenarios/qlib/experiment/factor_template` (`conf_baseline.yaml`, `conf_combined_factors.yaml`, `conf_combined_factors_sota_model.yaml`, `read_exp_res.py`).

### 3.5 Step `coding` (CoSTEER)

`RDLoop.coding` calls `self.coder.develop(prev_out["direct_exp_gen"]["exp_gen"])` and logs `sub_workspace_list`.

`QlibFactorCoSTEER` (`rdagent/scenarios/qlib/developer/factor_coder.py`) is the alias `QlibFactorCoSTEER = FactorCoSTEER`.

`FactorCoSTEER` (`rdagent/components/coder/factor_coder/__init__.py`):

- Settings: `FACTOR_COSTEER_SETTINGS` (`FactorCoSTEERSettings`, env prefix `FACTOR_CoSTEER_`).
- Evaluator: `CoSTEERMultiEvaluator(FactorEvaluatorForCoder(scen), scen)`.
- Strategy: `FactorMultiProcessEvolvingStrategy`.
- `super().__init__(..., evolving_version=2, scen=scen)`. `with_knowledge` stays the `CoSTEER` default `True`. `knowledge_self_gen` stays `True`.
- `develop` calls `CoSTEER.develop`, then copies `evolve_agent.evolving_trace[-1].feedback` onto `exp.prop_dev_feedback`.

`CoSTEER.develop` (`rdagent/components/coder/CoSTEER/__init__.py`) wraps the experiment as `EvolvingItem`, runs `RAGEvoAgent.multistep_evolve` up to `max_loop`, keeps the last acceptable fallback (`should_use_new_evo` is true when `new_fb.is_acceptable()`), and `_exp_postprocess_by_feedback` raises `CoderError` when every sub-task feedback is unacceptable.

`FactorMultiProcessEvolvingStrategy` (`rdagent/components/coder/factor_coder/evolving_strategy.py`) subclasses `MultiProcessEvolvingStrategy`. Implementation prompts are in `rdagent/components/coder/factor_coder/prompts.yaml`:

- `evolving_strategy_factor_implementation_v1_system` asks for JSON `{"code": "The Python code as a string."}`.
- `evolving_strategy_factor_implementation_v2_user` passes the task plus queried failures and similar successes.
- Evaluators: `evaluator_code_feedback_v1_*`, `evaluator_final_decision_v1_*`, `evolving_strategy_error_summary_v2_*`. The error-summary system text includes the note "When processing data, avoid time leakage."

Each sub-task workspace is a `FactorFBWorkspace`. `execute(data_type="Debug")` is the default. Version 1 links `FACTOR_CoSTEER_DATA_FOLDER_DEBUG` (`git_ignore_folder/factor_implementation_source_data_debug`) for `Debug`, and `FACTOR_CoSTEER_DATA_FOLDER` (`git_ignore_folder/factor_implementation_source_data`) otherwise. It runs `python factor.py` with `FACTOR_CoSTEER_PYTHON_BIN` (default `python`) and reads `result.h5`. The interface the model is told to follow is `qlib_factor_interface`: a `calculate_*` function, `__main__`, and a single-column frame indexed by `datetime` and `instrument`.

`FactorEvaluatorForCoder.evaluate` calls `implementation.execute()` with the default `Debug` data. `CoSTEERMultiEvaluator` sets `evo.sub_tasks[index].factor_implementation = True` when that task's merged `final_decision` is true (`rdagent/components/coder/CoSTEER/evaluators.py`). Feedback later treats `factor_implementation == False` as "hypothesis not verified".

`FactorEmptyError` / `CoderError` during `coding` or `running` are caught by `LoopBase._run_step`. The step result is set to `None`, `_EXCEPTION` is set, and the loop jumps to the `feedback` step (`skip_loop_error_stepname`).

### 3.6 Step `running`

`FactorRDLoop.running` → `QlibFactorRunner.develop` (`rdagent/scenarios/qlib/developer/factor_runner.py`).

`QlibFactorRunner` subclasses `CachedRunner`. `develop` is wrapped with `@cache_with_pickle(CachedRunner.get_cache_key, CachedRunner.assign_cached_result)`.

Behavior:

1. If `based_experiments[-1].result is None`, recursively `develop` that experiment first (the empty baseline, or the latest accepted one).
2. Build `run_env` from a fresh `FactorBasePropSetting()`: train/valid/test dates, plus `feature_names` / `feature_expressions` from `exp.base_features`.
3. When `based_experiments` is non-empty (the normal proposal path):
   - If more than one prior `QlibFactorExperiment` exists, `process_factor_data` on that list builds the SOTA panel (`rdagent/scenarios/qlib/developer/utils.py`).
   - `process_factor_data(exp)` executes each sub-workspace with data type `"All"` (full `data_folder`, not debug), and also executes `base_feature_codes` the same way.
   - Empty new panel → `FactorEmptyError`.
   - `deduplicate_new_factors`: per-datetime column correlation, drop a new column whose max correlation with any SOTA column is `>= 0.99`. All survivors dropped → `FactorEmptyError`.
   - Concatenate, write `combined_factors_df.parquet`.
   - If a previous `QlibModelExperiment` exists, inject its `model.py` and run `conf_combined_factors_sota_model.yaml`. Otherwise run `conf_combined_factors.yaml` (LightGBM).
4. When `based_experiments` is empty: parquet + `conf_combined_factors.yaml` if `base_feature_codes` is set, else `conf_baseline.yaml`.

`QlibFBWorkspace.execute` (`rdagent/scenarios/qlib/experiment/workspace.py`):

- `MODEL_CoSTEER_ENV_TYPE=docker` → `QTDockerEnv`.
- `conda` (default) → `QlibCondaEnv`.
- Runs `qrun {qlib_config_name}` with `run_env`, then `python read_exp_res.py`.
- Reads `qlib_res.csv` into a `Series` (`index_col=0`, first column) and returns `(series, filtered_stdout)`.
- Stdout kept for the experiment is the regex of PyTorch-style `Epoch... train/valid` lines. LightGBM logs usually do not match, so `exp.stdout` is often empty on the factor path.

`exp.result` is that series (`Experiment.result` is `running_info.result`). `exp.stdout` is the filtered log.

Importing `factor_runner` calls `pandarallel.initialize(verbose=1)` at import time.

### 3.7 Step `feedback`

`RDLoop.feedback`:

- If `_EXCEPTION` is set, it builds `HypothesisFeedback(reason=str(e), decision=False, code_change_summary="", acceptable=False)` and does not call the summarizer.
- Else `summarizer.generate_feedback(prev_out["running"], self.trace)`.
- Optional `_interact_feedback`.

`QlibFactorExperiment2Feedback.generate_feedback`:

- `current_result = exp.result`, `sota_result = exp.based_experiments[-1].result`.
- `process_results` keeps `IMPORTANT_METRICS`: `IC`, `1day.excess_return_with_cost.annualized_return`, `1day.excess_return_with_cost.max_drawdown`.
- Prompts: `scenarios.qlib.prompts:factor_feedback_generation.system` and `.user`.
- JSON keys read back: `Observations`, `Feedback for Hypothesis`, `New Hypothesis`, `Reasoning`, `Replace Best Result` (`yes`/`no` via `convert2bool` → `HypothesisFeedback.decision`).

The system prompt tells the model to set `Replace Best Result` to yes for any small improvement, and specifically when annualized return improves.

SOTA lookup used by the model path, `Trace.get_sota_hypothesis_and_experiment`, walks `hist` from the end and returns the latest node whose feedback is truthy (`decision` is true). The factor summarizer itself does not call that method. It compares against `based_experiments[-1]`, which `convert_response` built as the latest history node with a truthy feedback.

### 3.8 Step `record`

`RDLoop.record` calls `self.trace.sync_dag_parent_and_hist((exp, feedback), loop_idx)`.

`exp` is the first present of `prev_out["running"]`, `prev_out["coding"]`, or `prev_out["direct_exp_gen"]["exp_gen"]`.

`sync_dag_parent_and_hist` appends `dag_parent` (root `()` or a parent index; selection `(-1,)` means the latest node) and appends `hist`. `idx2loop_id` maps the new history index to the loop id.

`HypothesisFeedback.__bool__` is `decision`. A `False` decision stays in `hist` and is excluded from the next round's `based_experiments` list (`if t[1]`).

## 4. Requested locations

### 4.1 Proposal prompts and JSON

| Piece | Path | Key |
| --- | --- | --- |
| Shared hypothesis system/user | `rdagent/components/proposal/prompts.yaml` | `hypothesis_gen` |
| Shared task system/user | same file | `hypothesis2experiment` |
| Factor hypothesis schema | `rdagent/scenarios/qlib/prompts.yaml` | `factor_hypothesis_output_format` |
| Factor hypothesis rules | same file | `factor_hypothesis_specification` |
| Factor task schema | same file | `factor_experiment_output_format` |
| History rendering | same file | `hypothesis_and_feedback`, `last_hypothesis_and_feedback` |
| Feedback schema | same file | `factor_feedback_generation` |
| Scenario prose | `rdagent/scenarios/qlib/experiment/prompts.yaml` | `qlib_factor_*` |
| Coder JSON | `rdagent/components/coder/factor_coder/prompts.yaml` | `evolving_strategy_factor_implementation_v1_system` (`{"code": "..."}`) |

### 4.2 Task and workspace names

There is no class named `Workspace` in the qlib factor scenario beyond the core ABC. The concrete types are:

- Task: `FactorTask` (`rdagent/components/coder/factor_coder/factor.py`).
- Per-factor workspace: `FactorFBWorkspace` (same file). Writes `factor.py`, expects `result.h5`.
- Backtest workspace: `QlibFBWorkspace` (`rdagent/scenarios/qlib/experiment/workspace.py`).
- Experiment: `QlibFactorExperiment` (`rdagent/scenarios/qlib/experiment/factor_experiment.py`).
- Alias: `FactorExperiment = Experiment` in `factor_coder/factor.py`. `FeatureExperiment` is the same alias.

### 4.3 CoSTEER knowledge base read/write, and how to disable it

Read/write implementation: `CoSTEERRAGStrategy` in `rdagent/components/coder/CoSTEER/knowledge_management.py`.

- `load_or_init_knowledge_base(former_knowledge_base_path)`: if the path exists, `secure_pickle_load`. For `evolving_version == 2` the object must be a `CoSTEERKnowledgeBaseV2`. Otherwise it constructs an empty v2 base (factor coder passes `evolving_version=2`).
- `dump_knowledge_base`: if `dump_knowledge_base_path` is `None`, it logs a warning and skips. Otherwise it writes a pickle.
- `load_dumped_knowledge_base`: no-op with a log line when the path is missing.
- `CoSTEERRAGStrategyV2.query` fills a `CoSTEERQueriedKnowledgeV2` from `success_task_to_knowledge_dict` plus component / error / former-trace queries (`v2_query_component_limit`, `v2_query_error_limit`, `v2_query_former_trace_limit`).
- `generate_knowledge` updates that dict from the evolving trace.

Paths on `FactorCoSTEERSettings` (prefix `FACTOR_CoSTEER_`):

- `knowledge_base_path: str | None = None` → former pickle.
- `new_knowledge_base_path: str | None = None` → dump pickle.

`CoSTEER.__init__` passes those into `CoSTEERRAGStrategyV2`. `RAGEvoAgent.multistep_evolve` queries when `with_knowledge` is true, and on each step, when `knowledge_self_gen` is true, calls `load_dumped_knowledge_base`, `generate_knowledge`, `dump_knowledge_base`.

Disable behavior that exists today:

- Leave both paths unset (the default). Disk read and disk write are skipped. An in-memory v2 base is still created, and `knowledge_self_gen=True` still records successes for later iterations of the same `develop()` call.
- There is no `FactorCoSTEERSettings` flag for `with_knowledge` or `knowledge_self_gen`. `RDLoop` constructs the coder as `import_class(PROP_SETTING.coder)(scen)` and passes no extra kwargs.
- `MultiProcessEvolvingStrategy.evolve_iter` raises `ValueError` when `queried_knowledge is None`, and the message says to keep `with_knowledge=True`. Turning retrieval off by passing `with_knowledge=False` on a raw `CoSTEER` breaks factor coding.
- `FactorBasePropSetting.knowledge_base` is a different, unused hook. Setting `QLIB_FACTOR_KNOWLEDGE_BASE` does not touch CoSTEER.

A later subclass can pass `knowledge_self_gen=False` while leaving `with_knowledge=True`, so `query` still returns an object and the in-memory base stops growing. That subclass is not written in this phase. Do not edit `CoSTEER.__init__` to add a silent off switch.

Related sampler knobs, still on `CoSTEERSettings` / `FACTOR_CoSTEER_`: `v2_knowledge_sampler` (default `1.0`), `fail_task_trial_limit` (default 20), `enable_filelock`, `filelock_path`.

### 4.4 Backtest, metrics, and which split feeds feedback

Configs: `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml` (the normal factor file; siblings `conf_baseline.yaml` and `conf_combined_factors_sota_model.yaml` use the same split pattern).

| Piece | Where it is set |
| --- | --- |
| Instruments / benchmark | `csi300` / `SH000300` |
| Handler range | `train_start` through `test_end` |
| Qlib expressions | `Alpha158DL` `feature` lists filled from env `feature_expressions` and `feature_names` |
| Extra columns | `StaticDataLoader` on `combined_factors_df.parquet` |
| Label | `Ref($close, -2)/Ref($close, -1) - 1` named `LABEL0` |
| Label processors | `DropnaLabel`, `CSZScoreNorm` with `fields_group: label` |
| Dataset segments | train / valid / test from `FactorBasePropSetting` |
| Portfolio backtest window | `port_analysis_config.backtest.start_time` = `test_start`, `end_time` = `test_end`. Strategy `TopkDropoutStrategy` topk 50, n_drop 5 |
| Model | `LGBModel` unless a SOTA `QlibModelExperiment` replaced `model.py` |
| Records | `SignalRecord`, `SigAnaRecord(ana_long_short=False, ann_scaler=252)`, `PortAnaRecord` |

`read_exp_res.py` calls `qlib.init()` with no arguments (provider comes from `~/.qlib`), lists experiments through `qlib.workflow.R`, takes the recorder with the latest `end_time`, and writes `pd.Series(latest_recorder.list_metrics())` to `qlib_res.csv`. It also loads `portfolio_analysis/report_normal_1day.pkl` to `ret.parquet`. The metrics series is what becomes `exp.result`. The chart is logged and is not what `process_results` reads.

Which rows the agent sees:

- Feedback comparison uses the three `IMPORTANT_METRICS` above, taken from that full series. The backtest window in the yaml is the test segment, so the annualized-return and max-drawdown keys are test-window portfolio stats when qlib records them under those names.
- IC is produced by qlib `SigAnaRecord` from `SignalRecord`'s saved label. At the qlib commit pinned in `QlibCondaEnv.prepare` (`2fb9380`), `SignalRecord.generate_label` prepares `segments="test"`. The yaml does not override that segment. This smoke did not execute `qrun`, so the exact metric-key strings in `qlib_res.csv` were not observed on this machine.
- The next hypothesis prompt does not use `IMPORTANT_METRICS`. `hypothesis_and_feedback` and `last_hypothesis_and_feedback` index `experiment.result` at `IC`, `1day.excess_return_without_cost.annualized_return`, and `1day.excess_return_without_cost.max_drawdown`. Feedback's accept/reject text uses the `with_cost` names. Both slices are test-period numbers if both keys exist. A missing key raises inside `.loc` while rendering the next prompt.
- Valid-segment loss is not passed into `QlibFactorExperiment2Feedback`. The user prompt receives `combined_result` only. `last_hypothesis_and_feedback` includes `experiment.stdout`, which the workspace filters down to a PyTorch epoch regex that LightGBM does not match.
- LightGBM still uses the valid segment internally for its own early stopping. That number does not enter `HypothesisFeedback`.

`deduplicate_new_factors` correlates columns on every datetime present in the concatenated panel. `process_factor_data` does not slice to train or valid before that.

### 4.5 MLflow

Two separate uses:

1. Qlib's recorder, always on when `qrun` succeeds. `read_exp_res.py` imports `mlflow.tracking.MlflowClient` and reads metrics through `qlib.workflow.R`. This is the series stored on `exp.result`. `RD_AGENT_SETTINGS.enable_mlflow` does not gate it.
2. RD-Agent workflow tracking. `RDAgentSettings.enable_mlflow` defaults to `False` (`rdagent/core/conf.py`). When true, `rdagent/utils/workflow/tracking.py` imports mlflow and `WorkflowTracker.log_workflow_state` logs `loop_index`, `step_index`, a packed datetime, API fail counts, and timer remainders. It does not log IC or portfolio return.

### 4.6 Asyncio parallelism

`LoopBase.run` (`rdagent/utils/workflow/loop.py`) starts one `kickoff_loop` task and `get_max_parallel()` copies of `execute_loop`.

`RDAgentSettings.step_semaphore` defaults to `1`. `get_max_parallel` returns that int, or `max(values)` when it is a dict such as `{"coding": 3, "running": 2}`. `direct_exp_gen` will not start another proposal while unfinished loops are at that cap.

`get_semaphore` forces limit 1 for steps named `feedback` and `record`. Other steps use the semaphore.

`is_force_subproc` is true when `subproc_step` is true or `get_max_parallel() > 1`. Non-final steps then run via `ProcessPoolExecutor` on a deepcopy of the step function and `loop_prev_out`. The last step (`record`) always runs in-process.

A second knob, `RD_AGENT_SETTINGS.multi_proc_n` (default 1), is inside a step: CoSTEER's `multiprocessing_wrapper` and `process_factor_data`. It is not the loop semaphore.

`FactorRDLoop` does not define its own parallel policy.

### 4.7 Checkpoint and resume

`LOG_SETTINGS.trace_path` defaults to `log/{UTC timestamp}` (`rdagent/log/conf.py`, prefix `LOG_`).

After each successful step, `LoopBase._run_step` calls `dump(session_folder / f"{loop}/{stepIndex}_{stepName}")`. `dump` pickles the whole loop with `secure_pickle_dump`. `__getstate__` drops `queue`, `semaphores`, `_pbar`, and multiprocessing queues.

Resume:

- CLI: `rdagent fin_factor --path <file-or-log-dir>`. `--checkout` is the default.
- `LoopBase.load`: a directory loads the latest `*/*_*` file under `__session__`. `checkout=True` points the session folder back at that tree, truncates later step pickles, and truncates log storage at the loaded trace end. `checkout` as a path writes the new session under that path. `--no-checkout` keeps later files.
- `run()` sets `loop_idx` back to 0 and clears the queue. It does not clear `step_idx`. `kickoff_loop` only re-executes step 0 when `step_idx[li] == 0`, so a resumed loop continues at the stored step.
- The docstring on `factor.main` shows the fire form: `dotenv run -- python rdagent/app/qlib_rd_loop/factor.py $LOG_PATH/__session__/1/0_propose --step_n 1`.
- `withdraw_loop` reloads the earliest pickle of the previous loop directory. `FactorRDLoop` does not set `withdraw_loop_error`, so nothing on the factor path triggers it.
- Workspace file checkpoints (`FBWorkspace.create_ws_ckp` / `recover_ws_ckp`) are the CoSTEER fallback inside one `develop()` call. They are separate from the session pickle.

## 5. Leakage review of stock `fin_factor` feedback

The outer loop selects factors using the test window, then shows those test numbers to the next hypothesis.

1. Portfolio metrics in feedback are computed on `test_start`–`test_end` (`conf_*.yaml` backtest block). Accept/reject (`Replace Best Result` → `HypothesisFeedback.decision`) is defined in the prompt as annualized-return improvement on that comparison. Accepted experiments are concatenated into the SOTA factor library on later rounds (`based_experiments` filter plus `process_factor_data`). The test window is both the score and the selection set.
2. IC in `exp.result` is test-segment IC if the pinned qlib `SignalRecord.generate_label` behavior holds (`segments="test"` at `2fb9380`). The hypothesis templates also print test backtest rows (`excess_return_without_cost.*`) into the next proposal. Valid is not the feedback split.
3. The label is a forward return (`Ref($close, -2)/Ref($close, -1) - 1`). `CSZScoreNorm` is declared on the label group only. Feature cross-sectional normalization is not declared in the factor yaml. That does not stop a generated `factor.py` from using future rows.
4. Full-sample factor values. `daily_pv_all.h5` is `D.features(...).loc["2008-12-29":]` with no end date (`factor_data_template/generate.py`). The runner executes factor code with data type `"All"`. The model train segment is still the qlib train slice, but any implementation that shifts backward, uses a full-sample z-score, or ranks using future dates leaks into the train rows.
5. CoSTEER debug data overlaps the test window. `daily_pv_debug.h5` is `2018-01-01`–`2019-12-31`, first 100 instruments. Default test is `2017-01-01`–`2020-08-01`. `FactorEvaluatorForCoder` executes on `Debug`. The coder LLM therefore sees factor values from dates inside the test segment while it is still editing code. Ground-truth value checks (`FactorEqualValueRatioEvaluator` and neighbors in `eva_utils.py`) compare frames to each other. They do not compare to the label. The leakage here is test-period prices and the model's own outputs, visible during code search.
6. Dedup uses the unsliced panel, including test dates, at threshold 0.99. A factor can be dropped because it matches SOTA on the test dates. That decision never goes back to the hypothesis as a neighbor expression.
7. The scenario text does not forbid lookahead. `qlib_factor_interface` specifies the file contract. The only in-tree leakage instruction found on the coder path is one line in `evolving_strategy_error_summary_v2_system`, and it is shown when summarizing similar errors, not on every implementation call.
8. `base_features` expressions (`ALPHA20`) are evaluated by qlib's expression engine inside the handler. Those operators are point-in-time as defined by qlib. They share the handler's full `start_time`–`end_time` range and the same test backtest. Expression mode that reuses this path inherits the test-metric selection leak even when each expression is causal.
9. Exception feedback (`decision=False`) does not add a test score, but the failed experiment can still sit in `hist`. It is excluded from `based_experiments` because the feedback is falsy.

## 6. Drift versus `v0.8.0` (`274e274d5dbb72cc2ea139d1a7c93d73ce9b1198`)

Compared by reading that tag's copies of the factor loop files. Class path strings in `FactorBasePropSetting` are unchanged (`QlibFactorScenario`, `QlibFactorHypothesisGen`, `QlibFactorHypothesis2Experiment`, `QlibFactorCoSTEER`, `QlibFactorRunner`, `QlibFactorExperiment2Feedback`). `FactorTask` and `FactorFBWorkspace` keep the same names. `IMPORTANT_METRICS` is the same three keys. `env_type` default `"conda"` is already the v0.8.0 default. CoSTEER `knowledge_base_path` / `new_knowledge_base_path` default `None` in both.

| Area | `v0.8.0` | This `v1.0.0` tree |
| --- | --- | --- |
| `factor.main` | Typer `checkout` flag on the fire entry. Local name `model_loop`. No `base_features_path`, no `**kwargs` | Plain `checkout: bool`. CLI flag lives on `fin_factor_cli`. Adds `base_features_path` and interaction `kwargs`. Local name `factor_loop` |
| CLI registration | `app.command(name="fin_factor")(fin_factor)` passed `main` through directly | Wrapper `fin_factor_cli` |
| `FactorRDLoop.skip_loop_error` | `(FactorEmptyError,)` only | Also `CoderError`, and `skip_loop_error_stepname = "feedback"`. `LoopBase.skip_loop_error_stepname` is new |
| `RDLoop` size and steps | About 92 lines. Steps are `direct_exp_gen`, `coding`, `running`, `feedback`. `feedback` appends `trace.hist` itself. A comment claims `record` is hardcoded on `LoopBase`. `LoopBase` at that tag has no `record` method and `Trace` has no `sync_dag_parent_and_hist` | `record` is a real step and calls `Trace.sync_dag_parent_and_hist`. Exception feedback is a short `HypothesisFeedback` and is stored by `record`, not inside `feedback` |
| `direct_exp_gen` | Returns `{"propose", "exp_gen"}` only | Also copies `plan` features onto the experiment and the last based experiment |
| Base features | No `RDLoop.plan`, no `ALPHA20`, no `QlibFactorExperiment.base_features` | `plan["features"]` starts as `ALPHA20`. Runner passes `feature_names` / `feature_expressions` and the split dates into `qrun` env. `num_features` is `len(base_features) + columns` instead of `RD_AGENT_SETTINGS.initial_fator_library_size + columns` |
| User interaction | Not on `RDLoop` | `_interact_init_params`, `_interact_hypo`, `_interact_feedback` |
| Scenario experiment blurb | `qlib_factor_experiment_setting` rendered with no dates | Rendered with `FACTOR_PROP_SETTING` train/valid/test |
| Runner config choice | Inside the `based_experiments` branch, combined factors always use `conf_combined_factors.yaml` in the non-SOTA-model case. The empty-`based_experiments` branch calls `conf_baseline.yaml` (the ternary on `len(based_experiments)` is dead there because that branch already knows the list is empty) | Dates and base expressions always go out as env. Empty `based_experiments` plus `base_feature_codes` uses `conf_combined_factors.yaml`; otherwise `conf_baseline.yaml` |
| `exp.tasks = unique_tasks` | Present | Present. Still does not replace `sub_tasks` |
| Hypothesis `concise_*` | `convert_response` already `.get`s them while the JSON schema lists only `hypothesis` and `reason` | Same |

Items that matter for expression mode and were not on `v0.8.0`: the `base_features` expression dict, the qrun env injection of those expressions, the explicit `record` / DAG append, and `CoderError` skipping straight to feedback.

## 7. AlphaAgent ideas

Source inspected: `RndmVariableQ/AlphaAgent` default branch (`README.md`, `docs/operations_manual.md`, `alphaagent/factor/zoo/similarity.py`, `alphaagent/factor/ingest.py`, `alphaagent/factor/mining/prompts.py`, `alphaagent/dsl/core/parser.py`). No source was copied.

What that tree actually implements:

- **Similarity is value correlation, not an expression tree.** `SimilarityMatrix` records kind `cross_sectional_pearson_mean` between stored factor panels. Ingest rejects a candidate when the max absolute cross-sectional correlation against the library is at least `max_cs_corr`, and the mining prompt tells the agent to rewrite using `similarity.top_neighbors` (including the neighbor expression). That is crowding control in value space.
- **Complexity is not a scored constraint.** The mining prompt, ingest policy, and DSL parser do not apply a depth, node-count, or operator-budget penalty. The parser raises Python's recursion limit so deep nests can parse. That is a parser limit, not a research regularizer.
- **Hypothesis alignment is not a metric.** The miner is instructed to try several expressions as separate hypotheses on the train split, then a sparse validation check, and to call `submit_factor` only after train and validation gates (IC, ICIR, coverage, cross-sectional lag-1 autocorrelation, monthly IC sign share). Symbols named align in that repo (`align_series_to_panel`, `canonical_align`, `realign_factorlib_to_panel`) line factor rows up with the panel index after new dates are appended. They do not score whether an expression matches a written hypothesis.

Useful comparison with this RD-Agent tree, still without taking their code: `QlibFactorRunner.deduplicate_new_factors` already drops a new column at correlation `0.99`, but it does not return the colliding factor's expression to `QlibFactorHypothesisGen`, and it correlates the full date index, including the test window. AlphaAgent's ingest gate is the closer pattern if a later phase wants "too similar, here is the neighbor, rewrite", and their prompt keeps the accept gate on train plus a small validation check rather than on the test backtest.

An expression-tree distance, a complexity penalty, and a hypothesis-versus-expression judge are not in that repository. If those three were specified from an earlier AlphaAgent paper, they need a different source than this GitHub tree. They are not implemented here either.

## 8. Extension-point plan (no code)

The loop already selects implementations by class-path strings. A later phase can add new subclasses and point `QLIB_FACTOR_*` at them. This phase does not add those classes.

Existing switch, already live, no edit required to pick up a new class once it is importable:

| Env (prefix `QLIB_FACTOR_`) | Field | Stock class to subclass |
| --- | --- | --- |
| `QLIB_FACTOR_SCEN` | `scen` | `QlibFactorScenario` |
| `QLIB_FACTOR_HYPOTHESIS_GEN` | `hypothesis_gen` | `QlibFactorHypothesisGen` |
| `QLIB_FACTOR_HYPOTHESIS2EXPERIMENT` | `hypothesis2experiment` | `QlibFactorHypothesis2Experiment` |
| `QLIB_FACTOR_CODER` | `coder` | A new `Developer`. Do not subclass by editing `FactorCoSTEER` / `CoSTEER` |
| `QLIB_FACTOR_RUNNER` | `runner` | `QlibFactorRunner` |
| `QLIB_FACTOR_SUMMARIZER` | `summarizer` | `QlibFactorExperiment2Feedback` |

Suggested new modules, all under `rdagent/scenarios/qlib/`, so `rdagent/components/coder/CoSTEER/`, `rdagent/components/proposal/`, `rdagent/components/workflow/rd_loop.py`, and `rdagent/utils/workflow/loop.py` stay untouched. Names below are proposals for a later phase. They are not in the tree.

| Proposed type | Subclass of | Responsibility |
| --- | --- | --- |
| `ExprFactorScenario` | `QlibFactorScenario` | Describe an executable expression (qlib operators, the same shape as `ALPHA20`) instead of a Python `factor.py` contract. New prompt file, do not rewrite `experiment/prompts.yaml` in place if the Python path must keep working |
| `ExprFactorHypothesisGen` | `QlibFactorHypothesisGen` | `prepare_context` points at a schema whose JSON is still one hypothesis, and whose text asks for expressions rather than latex-only formulations |
| `ExprFactorHypothesis2Experiment` | `QlibFactorHypothesis2Experiment` | `convert_response` builds tasks and assigns `sub_tasks` (the stock `exp.tasks = ...` write is a no-op). Keep `based_experiments` construction or the runner's SOTA concat changes meaning |
| `ExprFactorTask` | `FactorTask` or `CoSTEERTask` | Add an expression string field. Leave `FactorTask.__init__` unchanged so pickle compatibility for old sessions stays intact |
| `ExprFactorExperiment` | `QlibFactorExperiment` | Same `base_features` dict the runner already injects into `feature_expressions` |
| `ExprFactorDeveloper` | `rdagent.core.developer.Developer` | Validate expressions and store them on the experiment. Do not call `FactorCoSTEER.develop`. `with_knowledge=False` on CoSTEER is not a viable shortcut (`evolve_iter` requires queried knowledge) |
| `ExprFactorRunner` | `QlibFactorRunner` | Put accepted expressions into `base_features` / the yaml feature list. Skip `FactorFBWorkspace.execute` and `combined_factors_df.parquet` for those factors. Keep date env from `FactorBasePropSetting` |
| `ExprFactorExperiment2Feedback` | `QlibFactorExperiment2Feedback` | Decide from a split that is not the test window (see open questions). New prompt keys, leave `factor_feedback_generation` in place for the Python path |
| `ExprFactorPropSetting` | `FactorBasePropSetting` | New module. Defaults the six class paths at the subclasses above. `FactorBasePropSetting` itself stays the stock `fin_factor` setting |

`FactorRDLoop` / `RDLoop` can stay. Their steps already match proposal → exp gen → coder → runner → feedback → record. A new loop subclass is only needed if expression mode changes step order.

CLI gap: `fin_factor_cli` always passes `FACTOR_PROP_SETTING` by constructing `FactorRDLoop` inside `main`. Env overrides of the six strings apply, because `FactorBasePropSetting()` reads the environment. A dedicated command or a `base_features_path` flag is a later edit to `rdagent/app/cli.py` and `factor.main`. Not in this phase.

Coder knowledge: disk KB stays off while the two `FACTOR_CoSTEER_*` paths are unset. Expression mode that never constructs `FactorCoSTEER` does not touch that KB.

## 9. Questions for the human

1. Expression language for phase 1: qlib operator strings in the same shape as `ALPHA20` (`rdagent/utils/qlib.py`), or a separate DSL? The runner env `feature_expressions` already feeds qlib `Alpha158DL`.
2. Should an expression factor skip CoSTEER entirely (section 8 `ExprFactorDeveloper`), or should CoSTEER remain for a thin syntax repair loop? `with_knowledge=False` currently crashes `MultiProcessEvolvingStrategy.evolve_iter`.
3. Feedback split: keep test-window IC and `excess_return_with_cost` (stock behavior), or score validation and leave the test window unread by `HypothesisFeedback`? `test_end` default `2020-08-01` is also near the end of the usual qlib cn dump, so a later holdout needs newer data.
4. `exp.tasks = unique_tasks` does not update `sub_tasks`, at `v0.8.0` and at `v1.0.0`. Confirm phase 1 should dedup by assigning `sub_tasks` in the new converter and leave `factor_proposal.py` unchanged.
5. `Hypothesis` requires `concise_*` fields that `factor_hypothesis_output_format` does not request. They are `None` today. `process_factor_data` uses `concise_justification` only as a log source name. Should expression mode stop requiring them?
6. A live `qrun` was not run. Please confirm on the pinned qlib (`2fb9380`) that `qlib_res.csv` contains both `1day.excess_return_with_cost.*` (feedback) and `1day.excess_return_without_cost.*` (next-round prompt). A missing key breaks `.loc` while rendering `hypothesis_and_feedback`.
7. AlphaAgent on GitHub does not contain expression-tree similarity, a complexity penalty, or a hypothesis-alignment score (section 7). If those three are required, point at the paper or the repo revision that has them.
8. Non-Docker smoke died before keys and before qlib. Is the intended dev box conda env `rdagent4qlib` plus a prebuilt `daily_pv.h5`, given that data bootstrap is hardcoded to `QTDockerEnv` and `make init-qlib-env` creates a different env name (`qlibRDAgent`, Python 3.8)?
9. Phase 1 interpreter: this host is Python 3.12 and the editable install imported. CI classifiers and constraint files are 3.10 and 3.11. There is no `constraints/3.12.txt`.
10. This clone has no git tags, so the installed distribution version is `0.1.dev1023` even though GitHub tag `v1.0.0` is this SHA. Say if phase 1 should fetch tags so setuptools-scm reports `1.0.0`.
11. `QLIB_FACTOR_EVOLVING_N` is unused. Coder loops follow `FACTOR_CoSTEER_MAX_LOOP`. Confirm expression mode should ignore `evolving_n`.
12. Dedup threshold `0.99` includes test dates and does not report the neighbor expression. Should expression mode move that check onto train/valid and pass the neighbor string back into hypothesis gen?
13. The log server passes `user_interaction_queues` into `fin_factor`. The CLI does not. Should expression mode support that IPC path in phase 1?
