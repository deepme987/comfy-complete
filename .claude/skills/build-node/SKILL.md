---
name: build-node
description: Scaffold a new ComfyUI custom node from a use case description, enforcing standard practices
---

# Build Custom Node

**Usage**: `/build-node <description of what the node should do>`

Scaffolds a complete, standards-compliant ComfyUI custom node pack from a
natural language description. The output is a working node that follows all
ComfyUI conventions and is ready for comfy-complete submission.

---

## Hard Rules (non-negotiable)

These rules are enforced automatically. Violations are errors, not warnings.

### R1: Node Class Structure
Every node class MUST have ALL of these:
```python
class MyNode:
    @classmethod
    def INPUT_TYPES(cls):
        return {"required": {...}, "optional": {...}}

    RETURN_TYPES = ("IMAGE",)       # Tuple of type strings
    RETURN_NAMES = ("image",)       # Tuple of output names
    FUNCTION = "execute"            # Method name to call
    CATEGORY = "pack-name/subcategory"

    def execute(self, **kwargs):
        ...
```

Missing any of these = broken node. No exceptions.

### R2: Output Node Contract
Any node that saves/previews images MUST:
- Set `OUTPUT_NODE = True`
- Return `{"ui": {"images": results_list}}` from its function
- Use `folder_paths.get_save_image_path()` for save paths

Cloud uploads rely on this. Empty `{"ui": {}}` = invisible output.

### R3: Model Loading via folder_paths
NEVER hardcode model paths. ALWAYS use:
```python
import folder_paths
model_path = folder_paths.get_full_path("checkpoints", model_name)
```

For model downloads, use environment detection:
```python
import comfy.utils
env = comfy.utils.get_execution_environment()
if model_path and os.path.exists(model_path):
    return load_model(model_path)
elif env == "local":
    return download_and_load(model_name)
else:
    raise RuntimeError(f"Model {model_name} not pre-provisioned")
```

### R4: Security Blockers
These patterns are NEVER acceptable. The node will be rejected:
- `eval()`, `exec()` — arbitrary code execution
- `os.system()`, `subprocess` with `shell=True` — command injection
- `pickle.loads()`, `pickle.load()` — deserialization attacks
- `torch.load()` without `weights_only=True` — pickle vulnerability
- Runtime `pip install` — breaks reproducibility
- Runtime model downloads without environment check — breaks cloud

### R5: Dependency Discipline
- Pin ALL dependencies to exact versions: `opencv-python-headless==4.10.0`
- Never depend on `numpy<2` (cloud uses numpy 2.2.6)
- Never override torch, torchvision, torchaudio, safetensors
- Prefer `opencv-python-headless` over `opencv-python` (no GUI needed)
- Zero dependencies = best case. Every dep is a compatibility risk.

### R6: Input Validation
- STRING inputs used as file paths MUST be validated
- Never pass user strings directly to `open()`, `os.path.join()`, or similar
- Use ComfyUI's built-in input types (IMAGE, LATENT, MODEL, etc.) over raw strings

### R7: No Global State
- No module-level mutable state (dicts, lists, caches)
- No `@lru_cache` on methods that take model paths
- Each execution must be independent — cloud runs nodes on separate pods

### R8: Standard Types Only
Prefer ComfyUI's built-in types for inputs/outputs:
- IMAGE, MASK, LATENT, MODEL, CLIP, VAE, CONDITIONING
- INT, FLOAT, STRING, BOOLEAN
- Custom types are allowed but require more testing and may not render in all UIs

---

## Scaffolding Process

### Step 1: Understand the Request

Parse the user's description to identify:
- **Core function**: What transformation does the node perform?
- **Input types**: What does it take in? (image, text, model, etc.)
- **Output types**: What does it produce?
- **Dependencies**: What libraries are needed? (torch, PIL, cv2, etc.)
- **Models**: Does it need pre-trained models?
- **Category**: Where does it fit in the ComfyUI menu?

### Step 2: Check for Duplicates

Before scaffolding, check if this functionality already exists:
```bash
# Check existing nodes in comfy-complete
rg 'class_type' tests/node-tests/ --type json | grep -i "<keyword>"

# Check ComfyUI built-in nodes
python -c "
import json
with open('object_info.json') as f:
    info = json.load(f)
for name, node in info.items():
    if '<keyword>' in name.lower() or '<keyword>' in node.get('description', '').lower():
        print(f'{name}: {node.get(\"description\", \"\")[:80]}')
"
```

If a duplicate exists, inform the user and suggest using the existing node instead.

### Step 3: Scaffold the Node Pack

Create the following structure:
```
<pack-name>/
    __init__.py          # NODE_CLASS_MAPPINGS, NODE_DISPLAY_NAME_MAPPINGS
    nodes.py             # Node class definitions
    requirements.txt     # Pinned dependencies (if any)
    LICENSE              # MIT or Apache-2.0 (REQUIRED)
    README.md            # Brief description
    pyproject.toml       # Package metadata
```

#### `__init__.py` template:
```python
"""<Pack description>."""

from .nodes import NODE_CLASS_MAPPINGS, NODE_DISPLAY_NAME_MAPPINGS

__all__ = ["NODE_CLASS_MAPPINGS", "NODE_DISPLAY_NAME_MAPPINGS"]
```

#### `nodes.py` template:
```python
"""<Pack description> - Node definitions."""

import torch
import numpy as np
from PIL import Image


class <NodeName>:
    """<One-line description>."""

    @classmethod
    def INPUT_TYPES(cls):
        return {
            "required": {
                # Use ComfyUI standard types
                "image": ("IMAGE",),
            },
            "optional": {},
        }

    RETURN_TYPES = ("IMAGE",)
    RETURN_NAMES = ("image",)
    FUNCTION = "execute"
    CATEGORY = "<pack-name>"

    def execute(self, image):
        # image is a torch tensor [B, H, W, C] in range [0, 1]
        # Process and return in same format
        return (image,)


NODE_CLASS_MAPPINGS = {
    "<NodeName>": <NodeName>,
}

NODE_DISPLAY_NAME_MAPPINGS = {
    "<NodeName>": "<Display Name>",
}
```

### Step 4: Apply Best Practices

For each node class, verify:
- [ ] INPUT_TYPES returns dict with "required" key
- [ ] RETURN_TYPES is a tuple (single element needs trailing comma)
- [ ] FUNCTION matches an actual method name
- [ ] CATEGORY is set and meaningful
- [ ] Image tensors are [B, H, W, C] float32 in [0, 1]
- [ ] Mask tensors are [B, H, W] float32 in [0, 1]
- [ ] Latent is {"samples": tensor} dict
- [ ] No hardcoded paths, no eval/exec, no global state
- [ ] Error messages are descriptive (users see them in the UI)

### Step 5: Generate Test Workflow

For each node class, create a test workflow in API format:
```json
{
  "1": {
    "class_type": "EmptyLatentImage",
    "inputs": {"width": 256, "height": 256, "batch_size": 1}
  },
  "2": {
    "class_type": "<NodeName>",
    "inputs": {"image": ["1", 0]}
  }
}
```

Save to `tests/node-tests/<pack-name>/test_<node_name>.json`.

Use small dimensions (256x256) and deterministic seeds for reproducibility.

### Step 6: Present to User

Show the user:
1. File tree of scaffolded node pack
2. Each file's content
3. Suggested labels based on code analysis
4. Suggested supported_nodes.yaml entry
5. Next steps: "Run `/validate-node <path>` to check, then `/submit-node` to prepare PR"

---

## Patterns for Common Node Types

### Image-to-Image Transform
```python
def execute(self, image):
    # image: [B, H, W, C] float32
    batch = image.numpy()
    results = []
    for i in range(batch.shape[0]):
        img = (batch[i] * 255).astype(np.uint8)
        pil_img = Image.fromarray(img)
        # ... transform ...
        result = np.array(pil_img).astype(np.float32) / 255.0
        results.append(result)
    return (torch.from_numpy(np.stack(results)),)
```

### Model Loader
```python
@classmethod
def INPUT_TYPES(cls):
    return {
        "required": {
            "model_name": (folder_paths.get_filename_list("checkpoints"),),
        }
    }

RETURN_TYPES = ("MODEL",)
FUNCTION = "load"
CATEGORY = "loaders"

def load(self, model_name):
    model_path = folder_paths.get_full_path("checkpoints", model_name)
    if not model_path:
        raise ValueError(f"Model not found: {model_name}")
    model = comfy.sd.load_checkpoint_guess_config(model_path)
    return model
```

### Save/Preview Node
```python
OUTPUT_NODE = True

def execute(self, images, filename_prefix="output"):
    results = []
    for image in images:
        # ... save logic ...
        results.append({
            "filename": filename,
            "subfolder": subfolder,
            "type": "output",
        })
    return {"ui": {"images": results}}
```

---

## Decision Points

If the user's request involves:
- **File system access** → Add ReadsArbitraryFile / WritesToDisk labels, use folder_paths
- **Network calls** → Add NetworkAccess label, use environment detection
- **External APIs** → Add RequiresExternalAPI label, suggest API key handling via env var
- **GPU operations** → Add RequiresGPU label, handle CPU fallback
- **Large outputs (video/audio)** → Add CreatesLargeOutputs label
- **Model downloads** → Declare in models: field, use environment detection fallback
- **Webcam/display** → Add RequiresWebcam/RequiresDisplay label, these won't work on cloud
