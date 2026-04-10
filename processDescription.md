````md
# RD-Agent + Qlib Factor Execution Chain

## 1. Shortest Main Path

```text
rdagent fin_factor
→ rdagent/app/cli.py
→ rdagent/app/qlib_rd_loop/factor.py: main()
→ create FactorRDLoop(FACTOR_PROP_SETTING)
→ asyncio.run(factor_loop.run(...))

→ RDLoop main workflow:
   direct_exp_gen
   → coding
   → running
   → feedback
   → record

→ when the factor workflow reaches `running`,
  it actually executes:
  FactorRDLoop.running()
  which calls:
  self.runner.develop(prev_out["coding"])

→ self.runner is dynamically injected in RDLoop.__init__
→ under the factor scenario, runner = QlibFactorRunner

→ QlibFactorRunner.develop()
  reads time parameters from conf.py
  builds `env_to_use`
  decides which YAML to execute:
  conf_baseline.yaml
  / conf_combined_factors.yaml
  / conf_combined_factors_sota_model.yaml

→ QlibFBWorkspace.execute()
  actually runs:
  qrun conf_xxx.yaml
  python read_exp_res.py

→ env.py
  injects `env_to_use` into the subprocess environment

→ qrun
  enters qlib.cli.run.run()

→ qlib/cli/run.py
  renders YAML with Jinja2 + os.environ
  then YAML.load() parses it into a config dict
  then executes:
  qlib.init(...)
  task_train(...)
  recorder.save_objects(...)

→ read_exp_res.py
  exports:
  qlib_res.csv
  ret.pkl
````

---

## 2. Where the Real Control Points Are

1. **Global factor configuration entry**

   * `rdagent/app/qlib_rd_loop/conf.py`

2. **Main workflow skeleton**

   * `rdagent/components/workflow/rd_loop.py`

3. **Factor-specific entrypoint**

   * `rdagent/app/qlib_rd_loop/factor.py`

4. **Where time ranges and feature metadata are inserted into the runtime environment**

   * `rdagent/scenarios/qlib/developer/factor_runner.py`

5. **Where `qrun` is actually triggered**

   * `rdagent/scenarios/qlib/experiment/workspace.py`

6. **Where environment variables are actually injected into subprocess / docker**

   * `rdagent/utils/env.py`

7. **Templates actually consumed by `qrun`**

   * `rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml`
   * `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml`
   * `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors_sota_model.yaml`

8. **Where final results are exported**

   * `rdagent/scenarios/qlib/experiment/factor_template/read_exp_res.py`

9. **Actual Qlib workflow source code**

   * `/home/timeg/dev/qlib/qlib/cli/run.py`

10. **`qrun` entry script**

    * `/home/timeg/miniconda3/envs/rdagent4qlib/bin/qrun`

---

## 3. Real Execution Order Starting from the Command Line

### Step 1: Run the command

```bash
rdagent fin_factor
```

### Step 2: CLI dispatch

**File:**
`rdagent/app/cli.py`

**What it does:**
Maps the command name `fin_factor` to `main()` inside:

`rdagent/app/qlib_rd_loop/factor.py`

**Conclusion:**

```text
rdagent fin_factor
→ factor.py:main()
```

---

## 4. What `factor.py` Actually Does

**File:**
`rdagent/app/qlib_rd_loop/factor.py`

**Core logic:**

1. Imports `FACTOR_PROP_SETTING`
2. Creates `FactorRDLoop(FACTOR_PROP_SETTING)`
3. Calls `asyncio.run(factor_loop.run(...))`

Also:

* `FactorRDLoop` overrides `running()`

That means:

Although the overall workflow skeleton comes from `RDLoop`, once the factor workflow reaches the `running` step, it executes the subclass version first:

* `FactorRDLoop.running()`

not the parent default:

* `RDLoop.running()`

This distinction is very important.

---

## 5. What `RDLoop` Actually Does

**File:**
`rdagent/components/workflow/rd_loop.py`

The source code confirms the following:

### 5.1 Dynamic component instantiation in `__init__`

It dynamically instantiates:

* `scen`
* `hypothesis_gen`
* `hypothesis2experiment`
* `coder`
* `runner`
* `summarizer`

The key line is:

```python
self.runner = import_class(PROP_SETTING.runner)(scen)
```

So:

* the runner is **not hardcoded**
* it is dynamically imported from the class path stored in `PROP_SETTING.runner`

### 5.2 RDLoop defines the core steps

Inside `RDLoop`, the core steps are:

* `async def direct_exp_gen(...)`
* `def coding(...)`
* `def running(...)`
* `def feedback(...)`
* `def record(...)`

### 5.3 Meaning of each step

#### `direct_exp_gen`

* generate a hypothesis
* convert it into an experiment
* attach `base_features` and `base_feature_codes`

#### `coding`

```python
self.coder.develop(prev_out["direct_exp_gen"]["exp_gen"])
```

#### `running`

```python
self.runner.develop(prev_out["coding"])
```

#### `feedback`

* if no exception: `summarizer.generate_feedback(...)`
* if exception exists: constructs `HypothesisFeedback(decision=False, reason=str(e), ...)`

#### `record`

* syncs `(exp, feedback)` into the trace DAG

### Conclusion

`RDLoop` provides the overall workflow skeleton:

```text
direct_exp_gen
→ coding
→ running
→ feedback
→ record
```

It does **not** decide which factor runner to use by itself.

It only dynamically assembles components based on `PROP_SETTING`.

---

## 6. Who the Real Factor Runner Is

**File:**
`rdagent/app/qlib_rd_loop/conf.py`

This file defines:

```python
class FactorBasePropSetting(BasePropSetting)
```

The most important part is:

```python
runner = "rdagent.scenarios.qlib.developer.factor_runner.QlibFactorRunner"
```

So inside the factor workflow:

```text
PROP_SETTING.runner
→ QlibFactorRunner
```

Combined with the line in `RDLoop.__init__`:

```python
self.runner = import_class(PROP_SETTING.runner)(scen)
```

the real conclusion is:

When the factor workflow reaches `running`, what actually executes is:

```text
QlibFactorRunner.develop(...)
```

---

## 7. What `QlibFactorRunner.develop()` Actually Does

**File:**
`rdagent/scenarios/qlib/developer/factor_runner.py`

This is one of the most important execution-control layers in the entire chain.

Its behavior can be divided into six parts.

### 7.1 If `based_experiments` exists and the last baseline has no result yet

```python
if exp.based_experiments and exp.based_experiments[-1].result is None:
    exp.based_experiments[-1] = self.develop(exp.based_experiments[-1])
```

**Meaning:**

If the current experiment depends on a previous baseline and that baseline has not been executed yet, it recursively runs the baseline first.

---

### 7.2 Re-read time parameters from `conf.py`

```python
fbps = FactorBasePropSetting()

env_to_use = {
    "PYTHONPATH": "./",
    "train_start": fbps.train_start,
    "train_end": fbps.train_end,
    "valid_start": fbps.valid_start,
    "valid_end": fbps.valid_end,
    "test_start": fbps.test_start,
    "feature_names": str(list(exp.base_features.keys())),
    "feature_expressions": str(list(exp.base_features.values())),
}
if fbps.test_end is not None:
    env_to_use.update({"test_end": fbps.test_end})
```

This proves that:

The actual `train/valid/test` values passed into `qrun` are **not decided by the YAML file itself**.

Instead, `factor_runner.py` reads them from `conf.py` and then injects them into `env_to_use`.

---

### 7.3 If `based_experiments` exists, process old factors and new factors together

* first retrieves historical SOTA factors
* then runs `process_factor_data(exp)` for newly generated factors

If the new factors are empty, it raises:

```python
FactorEmptyError("Factors failed to run on the full sample ...")
```

If the new factors are too similar to previous factors and become empty after deduplication, it raises:

```python
FactorEmptyError("highly similar to previous factors ...")
```

---

### 7.4 Save combined factors to `combined_factors_df.parquet`

```python
target_path = exp.experiment_workspace.workspace_path / "combined_factors_df.parquet"
combined_factors.to_parquet(target_path, engine="pyarrow")
```

This means:

The merged factor data is stored as **Parquet**, not Pickle.

---

### 7.5 Decide which YAML to run based on whether a SOTA model already exists

If a SOTA model experiment exists, it runs:

* `conf_combined_factors_sota_model.yaml`

Otherwise, it runs:

* `conf_combined_factors.yaml`

---

### 7.6 If there is no `based_experiments`, there are two branches

#### Case A: `exp.base_feature_codes` is not empty

* process factor data
* save `combined_factors_df.parquet`
* execute:

```text
conf_combined_factors.yaml
```

#### Case B: `exp.base_feature_codes` is empty

Directly execute:

```text
conf_baseline.yaml
```

Finally:

```python
result, stdout = exp.experiment_workspace.execute(...)
```

If `result is None`:

```python
raise FactorEmptyError(...)
```

Otherwise:

```python
exp.result = result
exp.stdout = stdout
return exp
```

---

## 8. `QlibFBWorkspace.execute()` Is the Real Place Where `qrun` Is Executed

**File:**
`rdagent/scenarios/qlib/experiment/workspace.py`

The core logic is:

```python
def execute(self, qlib_config_name="conf.yaml", run_env={}):
    if MODEL_COSTEER_SETTINGS.env_type == "docker":
        qtde = QTDockerEnv()
    elif MODEL_COSTEER_SETTINGS.env_type == "conda":
        qtde = QlibCondaEnv(conf=QlibCondaConf())
    else:
        return None, "Unknown environment type"

    qtde.prepare()

    execute_qlib_log = qtde.check_output(
        local_path=str(self.workspace_path),
        entry=f"qrun {qlib_config_name}",
        env=run_env,
    )

    execute_log = qtde.check_output(
        local_path=str(self.workspace_path),
        entry="python read_exp_res.py",
        env=run_env,
    )
```

Then it checks:

1. whether `ret.pkl` exists

   * if not, the run is considered failed

2. whether `qlib_res.csv` exists

   * if yes, it reads the metrics and returns them
   * if no, it returns `None`

It also extracts shorter training log fragments from the full `qrun` log using regex.

### Conclusion

The place where `qrun` is really executed is **not**:

* `factor.py`
* `conf.py`
* `factor_experiment.py`

It is:

* `workspace.py -> execute()`

---

## 9. How `env.py` Actually Passes Variables into `qrun`

**File:**
`rdagent/utils/env.py`

The source confirms:

1. `check_output(...)`
2. `check_output(...)` calls `run(...)`
3. `run(...)` calls `__run_with_retry(...)`
4. `__run_with_retry(...)` finally calls `_run(...)`

Inside the local / conda execution branch, the key code is:

```python
process = subprocess.Popen(
    entry,
    cwd=cwd,
    env={**os.environ, **env},
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    shell=True,
    bufsize=1,
    universal_newlines=True,
)
```

### Meaning

`run_env` is **not** passed as a normal Python function argument.

Instead, it is merged directly into the system environment and becomes visible inside the `qrun` subprocess as `os.environ`.

So the real chain is:

```text
factor_runner.py
→ env_to_use
→ workspace.execute(run_env=env_to_use)
→ env.py subprocess.Popen(env={**os.environ, **env})
→ qrun subprocess can see:
   train_start / train_end / ... / feature_names / feature_expressions
```

Also, `env.py` defines:

```python
class QlibCondaConf(CondaConf):
    conda_env_name = "rdagent4qlib"
    default_entry = "qrun conf.yaml"
```

This means:

Under the conda branch, the default environment is:

```text
rdagent4qlib
```

---

## 10. Where `qrun` Actually Goes

**File:**
`/home/timeg/miniconda3/envs/rdagent4qlib/bin/qrun`

Its entry logic is:

```python
#!/home/timeg/miniconda3/envs/rdagent4qlib/bin/python
import sys
from qlib.cli.run import run

if __name__ == '__main__':
    sys.argv[0] = sys.argv[0].removesuffix('.exe')
    sys.exit(run())
```

### Meaning

When the shell executes `qrun`, it actually enters:

```text
qlib.cli.run.run()
```

So `qrun` is not a black box.

It is fundamentally just a Python entry script.

---

## 11. How Qlib Actually Consumes the YAML

**File:**
`/home/timeg/dev/qlib/qlib/cli/run.py`

Its core logic is:

1. `render_template(config_path)`

   * read YAML template text
   * use Jinja2 to detect template variables
   * read their values from `os.environ`
   * render final YAML text

2. `yaml.load(rendered_yaml)`

   * parse the final YAML into a Python dict

3. `qlib.init(**config["qlib_init"])`

4. `recorder = task_train(config["task"], experiment_name=experiment_name)`

5. `recorder.save_objects(config=config)`

This is the final closure of the environment-variable chain:

```text
time settings in conf.py
→ factor_runner.py builds env_to_use
→ env.py injects them into os.environ
→ qlib.cli.run.render_template() reads from os.environ
→ {{ train_start }} / {{ train_end }} / {{ test_end }} are replaced
→ only then does the YAML become the final concrete config
```

---

## 12. What the YAML Templates Actually Are

**Files:**

* `rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml`
* `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml`
* `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors_sota_model.yaml`

You must remember this:

**The YAML files are not final configs.
They are templates.**

For example:

```yaml
start_time: {{ train_start }}
end_time: {{ test_end }}

train: [{{ train_start }}, {{ train_end }}]
valid: [{{ valid_start }}, {{ valid_end }}]
test: [{{ test_start }}, {{ test_end }}]

feature:
  - {{ feature_expressions }}
  - {{ feature_names }}
```

These values are not hardcoded.

They are rendered at runtime by `qrun` using `os.environ`.

So the true precedence chain is:

```text
conf.py
→ factor_runner.py
→ env_to_use
→ env.py
→ qrun / qlib.cli.run.py
→ YAML rendering
```

---

## 13. What `read_exp_res.py` Does

**File:**
`rdagent/scenarios/qlib/experiment/factor_template/read_exp_res.py`

It does the following:

1. find the latest recorder / experiment
2. export metrics to `qlib_res.csv`
3. export the backtest curve to `ret.pkl`

So the most common output artifacts are:

* `qlib_res.csv`
* `ret.pkl`

And `workspace.py` determines whether the current `qrun` execution succeeded by checking these files.

---

## 14. The Real Log Already Proves This Chain

The `excited-margarine.log` you provided directly shows:

1. first:

   ```text
   Start Loop 0, Step 0: direct_exp_gen
   ```

2. then:

   ```text
   Start Loop 0, Step 1: coding
   ```

3. then:

   ```text
   Start Loop 0, Step 2: running
   ```

This matches the RDLoop step definitions exactly.

The log also shows:

```text
Entry:
qrun conf_baseline.yaml

Env:
train_start: 2008-01-01
train_end: 2018-12-31
valid_start: 2019-01-01
valid_end: 2021-12-31
test_start: 2022-01-01
feature_names: [...]
feature_expressions: [...]
```

This directly proves that:

* the `running` step really does call `QlibFactorRunner`
* `QlibFactorRunner` really does inject time ranges and feature metadata into the environment
* `workspace.py` really does execute `qrun conf_baseline.yaml`
* `env.py` really does pass those variables into the subprocess environment

Later in the log it also explicitly states:

```text
Output has been saved to .../qlib_res.csv
```

So this chain is not speculation.

It is the actual behavior on your machine.

---

## 15. Final Closed Loop (Based on Real Source Code + Real Log)

1. Run:

   ```bash
   rdagent fin_factor
   ```

2. `cli.py` dispatches the command to:

   ```text
   rdagent/app/qlib_rd_loop/factor.py:main()
   ```

3. `factor.py` creates:

   ```text
   FactorRDLoop(FACTOR_PROP_SETTING)
   ```

   and then calls:

   ```python
   asyncio.run(factor_loop.run(...))
   ```

4. `RDLoop.__init__` dynamically injects:

   * `scen`
   * `hypothesis_gen`
   * `hypothesis2experiment`
   * `coder`
   * `runner`
   * `summarizer`

5. RDLoop starts the workflow:

   ```text
   direct_exp_gen
   → coding
   → running
   → feedback
   → record
   ```

6. `direct_exp_gen`

   * generates a hypothesis
   * converts it into an experiment
   * attaches `base_features` and `base_feature_codes`

7. `coding` executes:

   ```python
   self.coder.develop(prev_out["direct_exp_gen"]["exp_gen"])
   ```

8. `running`

   * under the factor workflow, the actual method executed is:

     ```text
     FactorRDLoop.running()
     ```

9. `FactorRDLoop.running()` executes:

   ```python
   self.runner.develop(prev_out["coding"])
   ```

10. `self.runner` comes from dynamic import inside `RDLoop.__init__`

11. Under the factor scenario:

```text
PROP_SETTING.runner
=
rdagent.scenarios.qlib.developer.factor_runner.QlibFactorRunner
```

12. `QlibFactorRunner.develop()`

* re-reads `FactorBasePropSetting()`
* builds `env_to_use`

13. `QlibFactorRunner.develop()`

* chooses one of:

  * `conf_baseline.yaml`
  * `conf_combined_factors.yaml`
  * `conf_combined_factors_sota_model.yaml`

14. `QlibFBWorkspace.execute()`
    actually runs:

```text
qrun conf_xxx.yaml
python read_exp_res.py
```

15. `env.py`
    merges `run_env` into:

```python
env={**os.environ, **env}
```

and injects it into the `qrun` subprocess

16. `qrun`
    enters:

```text
qlib.cli.run.run()
```

17. `qlib/cli/run.py`

* renders YAML using Jinja2 + `os.environ`
* parses it with `yaml.load(...)`
* runs:

  * `qlib.init(...)`
  * `task_train(...)`
  * `recorder.save_objects(...)`

18. `read_exp_res.py`
    exports:

* `qlib_res.csv`
* `ret.pkl`

19. `workspace.py`
    checks `ret.pkl` and `qlib_res.csv`

* if successful: returns metrics
* if failed: returns `None`

20. `QlibFactorRunner.develop()`
    writes results back into:

* `exp.result`
* `exp.stdout`

21. `feedback`
    calls:

```python
summarizer.generate_feedback(...)
```

22. `record`
    writes the result into the trace DAG

---

## 16. The Most Common Confusions

### 16.1 Who controls `train/valid/test`?

Not the YAML itself.

The real source is:

* `conf.py`

The real landing path is:

* `factor_runner.py → env_to_use → qrun render YAML`

---

### 16.2 Who actually executes `qrun`?

Not you manually.

Not `factor_experiment.py` either.

The real execution point is:

* `workspace.py -> execute()`

---

### 16.3 How does `env` get into `qrun`?

Not as a function argument.

It is injected as subprocess environment variables through:

```python
subprocess.Popen(env={**os.environ, **env})
```

---

### 16.4 Which `running()` is actually executed?

Under the factor workflow, the actual method is:

* `factor.py -> FactorRDLoop.running()`

not the parent default:

* `RDLoop.running()`

---

### 16.5 Are the YAML files final configs?

No.

They are templates.

They only become final after `qrun` renders them with values from `os.environ`.

---

### 16.6 What level is `generate.py`?

`factor_data_template/generate.py` belongs more to the data-template / data-preparation layer.

It is **not** the primary control point for the main `fin_factor` execution chain.

If the question is:

> Why does `fin_factor` run with this time range?

then the priority inspection path should be:

```text
conf.py
→ factor_runner.py
→ workspace.py
→ env.py
→ qrun
→ YAML
```

---

## 17. Most Important Source Code Locations

Use this order when debugging in the future.

### A. Command entry

`rdagent/app/cli.py`

### B. Factor entrypoint

`rdagent/app/qlib_rd_loop/factor.py`

### C. Factor global configuration

`rdagent/app/qlib_rd_loop/conf.py`

### D. Workflow skeleton

`rdagent/components/workflow/rd_loop.py`

### E. Factor runtime core

`rdagent/scenarios/qlib/developer/factor_runner.py`

### F. `qrun` trigger

`rdagent/scenarios/qlib/experiment/workspace.py`

### G. Environment injection

`rdagent/utils/env.py`

### H. Template source directory

`rdagent/scenarios/qlib/experiment/factor_template/`

### I. Result extraction

`rdagent/scenarios/qlib/experiment/factor_template/read_exp_res.py`

### J. `qrun` entry script

`/home/timeg/miniconda3/envs/rdagent4qlib/bin/qrun`

### K. Qlib workflow execution source

`/home/timeg/dev/qlib/qlib/cli/run.py`

---

## 18. One-Sentence Final Summary

`fin_factor` in RD-Agent does **not** simply "run a YAML file directly".

Its real execution chain is:

```text
conf.py provides global configuration
→ factor.py creates FactorRDLoop
→ RDLoop schedules direct_exp_gen / coding / running / feedback / record
→ running calls QlibFactorRunner.develop()
→ factor_runner.py builds env_to_use from time ranges and feature metadata
→ workspace.py actually executes qrun conf_xxx.yaml
→ env.py injects env_to_use into the subprocess environment
→ qrun enters qlib.cli.run.run()
→ qlib/cli/run.py renders YAML using os.environ
→ task_train runs the Qlib workflow
→ read_exp_res.py exports qlib_res.csv and ret.pkl
→ runner returns results
→ summarizer generates feedback
→ record writes everything into the trace
```

```
```
