# Framework

Use PyTorch as the deep learning framework with typed modules and explicit device management.

Use PyTorch for all deep learning work. Define models as `nn.Module` subclasses with fully typed `forward` methods. Manage device placement explicitly through a single device configuration rather than scattering `.to(device)` calls throughout the codebase.

## Rationale

PyTorch's eager execution model produces debuggable training code where standard Python tooling (debuggers, profilers, type checkers) works without special instrumentation. Typed `forward` signatures enable basedpyright to validate tensor shapes at the module boundary, catching dimension mismatches before runtime. Centralized device management prevents silent CPU/GPU misalignment that causes performance degradation or runtime errors in multi-device setups.

- [PyTorch Patterns](../reference/pytorch.md)
- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)

## Examples

**Correct**:
```python
import torch
from torch import Tensor, nn

class Classifier(nn.Module):
    def __init__(self, input_dim: int, num_classes: int) -> None:
        super().__init__()
        self.encoder = nn.Linear(input_dim, 128)
        self.head = nn.Linear(128, num_classes)

    def forward(self, x: Tensor) -> Tensor:
        features = torch.relu(self.encoder(x))
        return self.head(features)

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = Classifier(input_dim=768, num_classes=10).to(device)
```

**Incorrect**:
```python
class Classifier(nn.Module):
    def forward(self, x):  # Untyped forward signature
        x = x.cuda()       # Hardcoded device, no central management
        return self.head(self.encoder(x))
```

## Exceptions

Research prototyping in notebooks may relax type annotations on intermediate tensor operations. Projects requiring ONNX export for production inference use `torch.onnx.export`, whose default exporter is `torch.export`-based in current PyTorch; TorchScript (`torch.jit.script`) is deprecated and is not a new target. Complex binary dependencies (CUDA, cuDNN) may require Pixi alongside uv for environment management.

## Tooling

Install PyTorch through uv with the appropriate index for CUDA support. Run `basedpyright` on model modules to validate type annotations.
