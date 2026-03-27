---
name: validate-node
description: Run all validation checks on a custom node pack before submission
---

# Validate Custom Node

**Usage**: `/validate-node <path-to-node-pack-or-name>`

Runs the full validation suite against a custom node pack: security scan,
license check, label analysis, dependency compilation, test coverage, and
registry verification. Reports pass/fail for each gate with actionable
fix instructions.

---

## Hard Rules

### Gate 1: Security (BLOCKING)
These cause immediate rejection — no override:
- `eval()` / `exec()` — arbitrary code execution
- `os.system()` / `shell=True` — command injection
- `pickle.loads()` / `pickle.load()` — deserialization attack
- `torch.load()` without `weights_only=True` — pickle vulnerability
- Runtime pip installs (`subprocess.*pip`, `os.system.*pip`)

### Gate 2: License (BLOCKING)
- AGPL license on node or ANY transitive dependency = rejected
- GPL-3.0 = rejected (incompatible with most node licenses)
- InsightFace / DeepFace dependencies = rejected (non-commercial license)
- No license file = warning (strongly recommend MIT or Apache-2.0)

### Gate 3: Labels (BLOCKING)
- Every behavioral characteristic MUST be labeled
- Missing a label that code analysis detects = rejection
- Extra labels (conservative) = acceptable

### Gate 4: Test Coverage (BLOCKING)
- Every non-exempt node class MUST have at least one test workflow
- Exempt: nodes with Incompatible, RequiresWebcam, RequiresDisplay,
  RequiresClipboard, BrokenNode labels
- Test workflows must be valid API-format JSON
- Test workflows must reference the node's class_type

### Gate 5: Dependencies (WARNING)
- `numpy<2` requirement = BLOCKER (cloud uses numpy 2.2.6)
- torch/torchvision/torchaudio overrides = BLOCKER
- Version conflicts with existing requirements.txt = WARNING
- Unpinned versions = WARNING

---

## Validation Process

### Step 0: Locate the Node Pack

If a path is given, use it directly. If a name is given, check:
1. Is it cloned locally? Check `/tmp/nodes/<name>` or `<cwd>/<name>`
2. Is it on the registry? `curl -s "https://api.comfy.org/nodes/<name>"`
3. Is it a GitHub URL? Clone it.

```bash
# Registry lookup
REGISTRY_INFO=$(curl -s "https://api.comfy.org/nodes/$NODE_NAME")
REPO_URL=$(echo "$REGISTRY_INFO" | python3 -c "import sys,json; print(json.load(sys.stdin).get('repository',''))")
```

### Step 1: Security Scan

```bash
scripts/pr-review/security-scan.sh "$NODE_PATH"
```

This scans for:
- **BLOCKERS**: eval, exec, os.system, shell=True, pickle, torch.load unsafe
- **WARNINGS**: dynamic imports, network requests, file writes, GPU usage
- **DEPLOYMENT**: custom endpoints, stateful nodes, API keys

Report each finding with file:line reference.

If the script isn't available, run inline:
```bash
# Blockers
rg -n 'eval\s*\(' --type py "$NODE_PATH" | grep -v '#.*eval' | grep -v 'test'
rg -n 'exec\s*\(' --type py "$NODE_PATH" | grep -v '#.*exec' | grep -v 'test'
rg -n 'os\.system\s*\(' --type py "$NODE_PATH"
rg -n 'shell\s*=\s*True' --type py "$NODE_PATH"
rg -n 'pickle\.(loads?|dumps?)\s*\(' --type py "$NODE_PATH"
rg -n 'torch\.load\s*\(' --type py "$NODE_PATH" | grep -v 'weights_only'
```

### Step 2: License Check

```bash
python3 scripts/add-node/check-license.py "$NODE_PATH"
```

This checks:
- Node's own LICENSE file
- All pip dependencies (transitive)
- Model references (InsightFace, DeepFace blocklist)

If the script isn't available:
```bash
# Find license file
find "$NODE_PATH" -maxdepth 2 -iname 'LICENSE*' -o -iname 'COPYING*' | head -1

# Check for blocked dependencies
rg -i 'insightface|deepface|agpl' "$NODE_PATH/requirements.txt" "$NODE_PATH/setup.py" "$NODE_PATH/pyproject.toml" 2>/dev/null
```

### Step 3: Label Analysis

```bash
python3 scripts/add-node/suggest-labels.py "$NODE_PATH"
```

This analyzes Python source to detect 15 of 18 behavioral labels:
- ReadsArbitraryFile, WritesToDisk, CreatesLargeOutputs
- NetworkAccess, RequiresExternalAPI, Stateful, HasCustomEndpoints
- PathParsing, RequiresWebcam, RequiresDisplay, RequiresClipboard
- RequiresGPU, ExecutesArbitraryCode, RuntimeModelDownload, RuntimePipInstall

Three labels require human judgment: DuplicateOfCoreNode, Incompatible, BrokenNode.

Compare suggested labels against what's declared in `supported_nodes.yaml`
(if an entry exists). Flag any mismatches.

### Step 4: Dependency Check

```bash
python3 scripts/add-node/compile-deps.py "$NODE_PATH" --requirements requirements.txt
```

This checks:
- Blacklisted packages (torch, numpy<2, etc.)
- Version conflicts with existing pinned requirements
- Downgrade protection for protected packages

### Step 5: Test Coverage

```bash
python3 scripts/check_test_coverage.py --yaml supported_nodes.yaml --pack "$NODE_NAME"
```

Verifies that every non-exempt node class has at least one test workflow
in `tests/node-tests/<pack-name>/`.

Also validate test workflow structure:
```bash
python3 scripts/validate_test_workflows.py tests/node-tests/"$NODE_NAME"/
```

### Step 6: Registry Verification

```bash
curl -s "https://api.comfy.org/nodes/$NODE_NAME" -o /dev/null -w "%{http_code}"
```

- 200 = exists on registry (good)
- 404 = not on registry (warning — may need GitHub URL format)

### Step 7: YAML Entry Validation

If a `supported_nodes.yaml` entry exists for this pack:
```bash
python3 scripts/add-node/validate-entry.py --yaml supported_nodes.yaml --name "$NODE_NAME"
```

---

## Report Format

Present results as a table:

```
## Validation Report: <pack-name>

| Gate | Status | Details |
|------|--------|---------|
| Security | PASS/FAIL | N blockers, M warnings |
| License | PASS/FAIL | <license-type> |
| Labels | PASS/FAIL | N suggested, M declared, K missing |
| Dependencies | PASS/WARN | N conflicts |
| Test Coverage | PASS/FAIL | N/M nodes covered |
| Registry | PASS/WARN | Found/Not found |
| YAML Entry | PASS/SKIP | Valid/Missing |

### Security Findings
<details for each finding>

### Label Recommendations
<table of suggested vs declared labels>

### Dependency Warnings
<details for each conflict>

### Missing Test Coverage
<list of untested node classes>
```

### Verdict

- **READY**: All gates pass → "Run `/submit-node` to prepare your PR"
- **FIXABLE**: Warnings only → list specific fixes needed
- **BLOCKED**: Any gate fails → list blocking issues with fix instructions
