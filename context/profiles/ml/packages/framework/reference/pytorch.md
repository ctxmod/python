# PyTorch Patterns

PyTorch framework principles covering nn.Module design, device management, data loading, and distributed training patterns.

PyTorch applications organize around four concerns: model definition through `nn.Module` subclasses, data loading through `DataLoader` with typed collation, device management through centralized configuration, and distributed training through native parallelism primitives.

## Model Definition

Define all models as `nn.Module` subclasses. Type-annotate the `forward` method signature with tensor dimensions documented in the docstring. Keep `__init__` responsible for layer construction and `forward` responsible for the computation graph.

```python
import torch
from torch import Tensor, nn

class TransformerEncoder(nn.Module):
    def __init__(self, vocab_size: int, d_model: int, nhead: int, num_layers: int) -> None:
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.pos_encoding = nn.Parameter(torch.randn(1, 512, d_model))
        encoder_layer = nn.TransformerEncoderLayer(d_model, nhead, batch_first=True)
        self.encoder = nn.TransformerEncoder(encoder_layer, num_layers)

    def forward(self, token_ids: Tensor) -> Tensor:
        """Encode token IDs. Input: (batch, seq_len). Output: (batch, seq_len, d_model)."""
        x = self.embedding(token_ids) + self.pos_encoding[:, :token_ids.size(1)]
        return self.encoder(x)
```

Compose complex architectures from smaller modules, where each sub-module handles one transformation. This enables independent testing and selective parameter freezing during fine-tuning. Accept sub-modules through constructors rather than constructing them internally.

### Module Guidelines

| Guideline | Rationale |
|-----------|-----------|
| Type `forward` parameters and return | basedpyright validates call sites |
| Document tensor shapes in docstrings | Shape mismatches are the most common bug |
| Accept sub-modules through constructor | Enables composition and testing |
| Avoid in-place operations in `forward` | Breaks autograd computation graph |
| Keep `forward` deterministic at eval time | Use `self.training` flag for dropout/noise |

## Data Loading

Define datasets as `torch.utils.data.Dataset` subclasses with typed `__getitem__` methods. Use custom `collate_fn` functions for variable-length sequences.

```python
from torch.utils.data import Dataset
from dataclasses import dataclass

@dataclass
class TextSample:
    token_ids: Tensor
    label: int

class TextDataset(Dataset[TextSample]):
    def __init__(self, texts: list[str], labels: list[int], tokenizer: Tokenizer) -> None:
        self.texts = texts
        self.labels = labels
        self.tokenizer = tokenizer

    def __len__(self) -> int:
        return len(self.texts)

    def __getitem__(self, idx: int) -> TextSample:
        ids = self.tokenizer.encode(self.texts[idx])
        return TextSample(token_ids=torch.tensor(ids), label=self.labels[idx])
```

### DataLoader Configuration

| Parameter | Recommendation | Rationale |
|-----------|---------------|-----------|
| `num_workers` | 4-8 per GPU | Overlap data loading with training |
| `pin_memory` | `True` for GPU training | Faster host-to-device transfer |
| `persistent_workers` | `True` if `num_workers > 0` | Avoids worker re-initialization |
| `drop_last` | `True` for training | Prevents batch norm issues with small final batch |

## Device Management

Centralize device selection in a single function. Pass the device to model and data operations rather than calling `.cuda()` on individual tensors.

```python
def get_device() -> torch.device:
    if torch.cuda.is_available():
        return torch.device("cuda")
    if torch.backends.mps.is_available():
        return torch.device("mps")
    return torch.device("cpu")
```

Avoid `tensor.cuda()` scattered through the codebase. It hardcodes GPU assumption and fails on CPU-only machines or Apple Silicon (MPS).

## Distributed Training

For multi-GPU training, use `DistributedDataParallel` (DDP). DDP replicates the model across GPUs and synchronizes gradients during backward passes, providing near-linear scaling.

| Strategy | GPUs | Use Case |
|----------|------|----------|
| Single GPU | 1 | Development, small models |
| DataParallel | 2-4 (single node) | Quick multi-GPU, simple setup |
| DistributedDataParallel | 2+ (any topology) | Production training, best performance |
| FSDP | 8+ | Models exceeding single GPU memory |

Prefer DDP over `DataParallel`. DataParallel uses a single-process multi-thread approach constrained by Python's GIL. DDP uses multiple processes with NCCL communication, achieving better throughput.

## Mixed Precision Training

Use `torch.amp` for automatic mixed precision to reduce memory usage and increase throughput:

```python
from torch.amp import autocast, GradScaler

scaler = GradScaler()
for batch in dataloader:
    optimizer.zero_grad()
    with autocast(device_type="cuda"):
        loss = criterion(model(batch.inputs), batch.labels)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

Mixed precision keeps master weights in FP32 while computing forward and backward passes in FP16/BF16. This halves memory for activations and gradients while maintaining training stability through loss scaling.

## Evidence

PyTorch patterns follow official PyTorch documentation and established practices from major ML research organizations. DDP scaling benchmarks show 90%+ efficiency on 8-GPU configurations with NCCL backend. Mixed precision reduces memory usage by 40-50% with minimal accuracy impact.

## Related

- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [Project Structure](../../../../core/packages/structure/reference/structure.md)
- [Training, Checkpointing, and Resumption](../../training/reference/training.md)
- [Experiment Design and Tracking](../../experiments/reference/experiments.md)
