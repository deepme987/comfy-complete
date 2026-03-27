---
name: submit-node
description: Prepare a supported_nodes.yaml entry and PR with all required fields for comfy-complete submission
---

# Submit Custom Node

**Usage**: `/submit-node <pack-name-or-path>`

Prepares a complete comfy-complete submission: generates the
`supported_nodes.yaml` entry, ensures test workflows exist, runs final
validation, and creates a PR-ready branch. This is the last step after
`/build-node` and `/validate-node`.

---

## Hard Rules

### R1: Registry Name Verification
ALWAYS verify the exact registry name before setting `name:`:
```bash
curl -s "https://api.comfy.org/nodes/<name>" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('id','NOT FOUND'))"
```
Wrong name = node won't install. No guessing.

### R2: Version Pinning
- Registry packs: use exact version string from registry API
- GitHub packs: use full 40-char commit SHA, never a branch name
```yaml
# Registry
- name: comfyui-example
  version: "1.2.3"

# GitHub (pinned to commit)
- name: https://github.com/author/repo@abc123def456...
  version: "1.0.0"
```

### R3: Alphabetical Ordering
The `node_packs:` list in `supported_nodes.yaml` MUST stay sorted
alphabetically by `name:`. Insert the new entry in the correct position.

### R4: Complete Label Coverage
Every label suggested by `scripts/add-node/suggest-labels.py` MUST be
included. Missing a label = CI rejection. Extra labels (conservative) are OK.

### R5: Dependency Overrides Only When Needed
Only add `dependency_overrides:` if the node needs packages NOT in the
base `requirements.txt` or needs a DIFFERENT version. Check first:
```bash
grep -i "<package>" requirements.txt
```
If already pinned at a compatible version, don't override.

### R6: Model Declaration Completeness
If the node uses models, ALL models must be declared with:
- `name:` — filename as it appears in the model directory
- `url:` — direct download URL (HuggingFace resolve links preferred)
- `directory:` — ComfyUI model subdirectory (must match folder_paths)
- `filename:` — (optional) if different from name

### R7: Test Workflow Existence
Every non-exempt node class MUST have at least one test workflow in
`tests/node-tests/<pack-name>/`. No tests = no merge.

### R8: No Scope Creep
This skill creates ONE PR for ONE node pack. Don't modify other packs,
don't update requirements.txt globally, don't change config files.

---

## Submission Process

### Step 1: Gather Information

Collect from the node pack:
```bash
NODE_NAME="<pack-name>"
NODE_PATH="<path-to-cloned-repo>"

# Get version from registry
VERSION=$(curl -s "https://api.comfy.org/nodes/$NODE_NAME" | python3 -c "import sys,json; print(json.load(sys.stdin).get('latest_version',{}).get('version',''))")

# Discover node classes
rg 'NODE_CLASS_MAPPINGS\s*=\s*\{' --type py "$NODE_PATH" -A 20 | grep '"'

# Check for web directory
ls "$NODE_PATH/web/" "$NODE_PATH/js/" 2>/dev/null && echo "HAS_WEB=true"

# Check for models
rg -n 'from_pretrained|hf_hub_download|snapshot_download|torch\.hub' --type py "$NODE_PATH"
```

### Step 2: Run Full Validation

Run `/validate-node` first (or the equivalent checks). Do NOT proceed if
any blocking gate fails.

```bash
# Quick validation
python3 scripts/add-node/validate-entry.py --yaml supported_nodes.yaml --name "$NODE_NAME" 2>&1
```

### Step 3: Generate YAML Entry

Build the `supported_nodes.yaml` entry:

```yaml
  - name: <registry-name>
    version: "<exact-version>"
```

Add optional fields ONLY if needed:

```yaml
    # Only if node has web/js directory for custom UI
    web_directory: <dirname>

    # Only if specific nodes need behavioral labels
    node_labels:
      NodeClassName:
        - LabelName
      AnotherNode:
        - ReadsArbitraryFile
        - NetworkAccess

    # Only if node needs packages not in base requirements.txt
    dependency_overrides:
      - "special-lib==2.3.1"

    # Only if node has system-level dependencies
    system-dependencies:
      - ffmpeg

    # Only if node uses downloadable models
    models:
      - name: "model-file.safetensors"
        url: "https://huggingface.co/author/repo/resolve/main/model-file.safetensors"
        directory: "checkpoints"
```

### Step 4: Insert Entry

Find the correct alphabetical position in `supported_nodes.yaml`:

```bash
# Show surrounding entries to find insertion point
grep '  - name:' supported_nodes.yaml | sort | grep -B2 -A2 "$NODE_NAME"
```

Use the Edit tool to insert at the correct position.

### Step 5: Ensure Test Workflows

Verify test workflows exist:
```bash
ls tests/node-tests/"$NODE_NAME"/ 2>/dev/null
```

If missing, create them. Each test workflow must:
- Be valid API-format JSON
- Reference the node's `class_type`
- Use small inputs (256x256 images, low step counts)
- Use deterministic seeds
- Include at least one output/assertion node

Template for a basic node test:
```json
{
  "1": {
    "class_type": "EmptyLatentImage",
    "inputs": {
      "width": 256,
      "height": 256,
      "batch_size": 1
    }
  },
  "2": {
    "class_type": "VAEDecode",
    "inputs": {
      "samples": ["1", 0],
      "vae": ["3", 2]
    }
  },
  "3": {
    "class_type": "CheckpointLoaderSimple",
    "inputs": {
      "ckpt_name": "v1-5-pruned-emaonly.safetensors"
    }
  },
  "4": {
    "class_type": "<TargetNodeClass>",
    "inputs": {
      "image": ["2", 0]
    }
  }
}
```

### Step 6: Create Branch and Commit

```bash
git checkout -b "feat/add-$NODE_NAME"
git add supported_nodes.yaml tests/node-tests/"$NODE_NAME"/
git commit -m "feat(custom-nodes): add $NODE_NAME"
```

### Step 7: Run Final CI Checks Locally

```bash
# YAML lint
python3 -c "import yaml; yaml.safe_load(open('supported_nodes.yaml'))" && echo "YAML: OK"

# Pytest
python3 -m pytest tests/ -v

# Validate entry
python3 scripts/add-node/validate-entry.py --yaml supported_nodes.yaml --name "$NODE_NAME"

# Test coverage
python3 scripts/check_test_coverage.py --yaml supported_nodes.yaml --pack "$NODE_NAME"

# Validate test workflows
python3 scripts/validate_test_workflows.py tests/node-tests/"$NODE_NAME"/
```

All must pass before creating PR.

### Step 8: Create PR

```bash
git push -u origin "feat/add-$NODE_NAME"

gh pr create \
  --title "feat(custom-nodes): add $NODE_NAME" \
  --body "$(cat <<'PREOF'
## Summary

**Node pack**: <name> v<version>
**Registry**: https://api.comfy.org/nodes/<name>
**Repository**: <github-url>
**Description**: <one-line description>

## Node Classes

| Class | Labels | Test |
|-------|--------|------|
| NodeName | label1, label2 | test_basic.json |

## Dependencies

<list dependency_overrides or "None">

## Models

<list models or "None">

## Checklist

- [ ] Tests pass (`pytest tests/ -v`)
- [ ] Labels correctly set in supported_nodes.yaml
- [ ] All dependencies pinned to exact versions
- [ ] Test workflows cover all non-exempt nodes

---

*Prepared with `/submit-node`*
PREOF
)"
```

### Step 9: Post-PR Summary

Tell the user:
1. PR URL
2. Which CI checks will run (validate, yaml-lint, test-workflows, ai-review)
3. What happens next (maintainer review → ephemeral test → merge)
4. Estimated timeline based on current queue

---

## Edge Cases

### GitHub URL format (not on registry)
```yaml
- name: https://github.com/author/repo@<full-commit-sha>
  version: "1.0.0"
```
The name IS the URL. Version is informational.

### Node with web directory
Check `web/` or `js/` in the repo root. If present:
```yaml
    web_directory: comfyui  # or whatever the directory is named
```

### Node with system dependencies
If the node requires system packages (ffmpeg, libgl1, etc.):
```yaml
    system-dependencies:
      - ffmpeg
```
These must be pre-installed in the Docker image.

### Large pack (50+ nodes)
For packs with many nodes:
- Group test workflows by functionality (not one per node)
- Chain related nodes in single workflow (tests multiple at once)
- Focus test effort on unique/complex nodes first
- Add coverage incrementally — partial is better than none
