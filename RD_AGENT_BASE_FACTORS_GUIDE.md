
---

```md
# RD-Agent base_factors.json — Final Guide

## 1. One-Line Conclusion

The directory used to read `base_factors.json` is **not hardcoded** in `rd_loop.py`.  
It is determined by `base_features_path`, which is passed from the Web UI backend.  

In the current UI workflow, `base_features_path` is set to the upload directory `trace_files_path`.

---

## 2. Key File Paths

### Feature Loader
```

/home/timeg/RD-Agent/rdagent/components/workflow/rd_loop.py

```

### fin_factor Entry
```

/home/timeg/RD-Agent/rdagent/app/qlib_rd_loop/factor.py

```

### Web UI Backend (Source of Truth)
```

/home/timeg/RD-Agent/rdagent/log/ui/app.py

```

### UI Config (Trace Root)
```

/home/timeg/RD-Agent/rdagent/log/ui/conf.py

````

---

## 3. Core Logic

### A. Feature Loading (`rd_loop.py`)

```python
def _init_base_features(self, base_features_path: str | None):
    if base_features_path is not None:
        base_dir = Path(base_features_path)
        base_factors_file = base_dir / "base_factors.json"

        feature_codes: dict[str, str] = {}
        for py_file in sorted(base_dir.glob("*.py")):
            feature_codes[py_file.name] = py_file.read_text()

        self.plan["feature_codes"] = feature_codes

        if not base_factors_file.exists():
            logger.info("No base_factors.json found")
        else:
            with base_factors_file.open("r") as f:
                features = json.load(f)
````

**Meaning:**

* Whatever path is passed → it reads from that directory
* It loads both:

  * `base_factors.json`
  * all `*.py` files

---

### B. factor.py (Pass-through Only)

```python
def main(..., base_features_path: Optional[str] = None):
    factor_loop._init_base_features(base_features_path)
```

**Meaning:**

* Does NOT determine the path
* Simply forwards it

---

### C. app.py (Actual Source of Path)

```python
log_folder_path = Path(UI_SETTING.trace_folder).absolute()

trace_files_path = log_folder_path / "uploads" / scenario / trace_name
```

```python
if scenario == "Finance Data Building":
    kwargs = {
        "loop_n": loop_n_val,
        "all_duration": all_duration_val,
        "base_features_path": str(trace_files_path),
    }
```

**Meaning:**

* Uploaded files are saved into `trace_files_path`
* This path is passed into RD-Agent as `base_features_path`

---

### D. conf.py (Trace Root)

```python
trace_folder: str = "./git_ignore_folder/traces"
```

**Resolved to:**

```
/home/timeg/RD-Agent/git_ignore_folder/traces
```

---

## 4. Final Paths

### Upload Directory (Used by base_features_path)

```
/home/timeg/RD-Agent/git_ignore_folder/traces/uploads/Finance Data Building/tranquil-key
```

### Windows Path

```
\\wsl.localhost\Ubuntu\home\timeg\RD-Agent\git_ignore_folder\traces\uploads\Finance Data Building\tranquil-key
```

### Log Directory (Different!)

```
/home/timeg/RD-Agent/git_ignore_folder/traces/Finance Data Building/tranquil-key
```

### Difference

| Path                 | Purpose     |
| -------------------- | ----------- |
| `traces/uploads/...` | input files |
| `traces/...`         | logs        |

---

## 5. Why `uploads` Sometimes Does Not Exist

From `app.py`:

```python
for file in files:
    if file:
        if not p.exists():
            p.mkdir(parents=True, exist_ok=True)
        file.save(target_path)
```

**Conclusion:**

* No upload → no `uploads/` directory
* Upload exists → directory is created

---

## 6. Correct Usage (Web UI)

1. Open:

```
http://127.0.0.1:19899
```

2. Select:

```
Finance Data Building
```

3. Upload:

```
base_factors.json
```

4. Click Start

---

## 7. What Actually Happens

```
Upload base_factors.json
→ request.files contains file
→ app.py creates uploads directory
→ file.save(...)
→ base_features_path = uploads path
→ rd_loop.py loads base_factors.json
```

---

## 8. Final Takeaway

The simplest way to make RD-Agent load your factors:

```
Upload base_factors.json via Web UI
```

No code modification needed.

---

## 9. base_factors.json Format

### Minimal Working Example

```json
{
  "MOM_10D": "$close / Ref($close, 10) - 1",
  "MOM_20D": "$close / Ref($close, 20) - 1",
  "VOL_MOM_10D": "$volume / Ref($volume, 10) - 1"
}
```

---

## 10. Format Rules

* Must be a JSON object
* Key = factor name
* Value = QLib expression string

---

### ❌ Invalid Example

```json
{
  "MOM_10D": {
    "expression": "..."
  }
}
```

---

### ✅ Valid Example

```json
{
  "MOM_10D": "$close / Ref($close, 10) - 1"
}
```

---

## 11. Supported QLib Expressions

### Momentum

```
$close / Ref($close, N) - 1
```

### Mean Reversion

```
Ref($close, N) / $close - 1
```

### Moving Average Deviation

```
$close / Mean($close, N) - 1
```

### Volatility

```
Std($close, N) / $close
```

### Volume Momentum

```
$volume / Ref($volume, N) - 1
```

### Price-Volume Correlation

```
Corr($close, $volume, N)
```

---

## 12. Final Summary

* `base_factors.json` is loaded from `base_features_path`
* Web UI sets this path automatically
* Uploading the file is the simplest and correct workflow

```
