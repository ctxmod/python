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
| DistributedDataParallel (`torchrun`) | 2+ (any topology) | Model fits on one GPU; the default for multi-GPU training |
| FSDP2 (`torch.distributed.fsdp.fully_shard`) | 2+ | Parameters, gradients, or optimizer state exceed one GPU's memory |

Do not use `DataParallel`: it is a single-process multi-thread approach constrained by Python's GIL and is slower than DDP even on one node. DDP uses one process per GPU with NCCL communication. For sharding, use FSDP2's per-parameter `fully_shard(module)` API, which converts parameters to `DTensor` in place; the wrapper-class `FullyShardedDataParallel` (FSDP1) remains for existing code, and the PyTorch docs direct FSDP1 users to the FSDP2 migration guide.

## Mixed Precision Training

Use `torch.amp` for automatic mixed precision to reduce memory usage and increase throughput:

```python
from torch.amp import autocast, GradScaler

# float16 needs loss scaling; bfloat16 (Ampere and newer) has float32's exponent range and does not.
use_fp16 = not torch.cuda.is_bf16_supported()
amp_dtype = torch.float16 if use_fp16 else torch.bfloat16
scaler = GradScaler("cuda", enabled=use_fp16)

for batch in dataloader:
    optimizer.zero_grad(set_to_none=True)
    with autocast(device_type="cuda", dtype=amp_dtype):
        loss = criterion(model(batch.inputs), batch.labels)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

Mixed precision keeps master weights in FP32 while computing forward and backward passes in FP16/BF16. This halves memory for activations and gradients. Loss scaling exists only for FP16's narrow exponent range; with BF16 the scaler is disabled and the same loop runs unchanged.

## Compilation and Export

`torch.compile(model)` is the optimization path for training and inference; compile once after moving the model to its device, and keep a non-compiled path for debugging. For deployment artifacts, `torch.export.export` produces an ahead-of-time graph, and `torch.onnx.export` uses the `torch.export`-based exporter by default (`dynamo=True` in PyTorch 2.15). TorchScript is deprecated: `torch.jit.script` emits "`torch.jit.script` is deprecated. Please switch to `torch.compile` or `torch.export`." (verified on 2.15.0), so new code does not target it.

## Checkpoint Loading Safety

`torch.load` defaults to `weights_only=True` since PyTorch 2.6: unpickling is restricted to tensors, primitive types, containers, and types registered through `torch.serialization.add_safe_globals()`. A checkpoint dictionary of state dicts, epoch counters, losses, and RNG states loads under the default — verified on 2.15.0 for model, AdamW optimizer, scheduler, and RNG state. Passing `weights_only=False` re-enables arbitrary code execution from the file; reserve it for a trusted legacy checkpoint that pickles whole objects, and migrate such checkpoints to state dicts. For weights exchanged with other teams or downloaded, prefer `safetensors`, which carries no executable payload.

## Evidence

PyTorch patterns follow official PyTorch documentation (2.15) and established practices from major ML research organizations. DDP scaling benchmarks show 90%+ efficiency on 8-GPU configurations with NCCL backend. Mixed precision reduces memory usage by 40-50% with minimal accuracy impact.

## Related

- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [Project Structure](../../../../core/packages/structure/reference/structure.md)
- [Training, Checkpointing, and Resumption](../../training/reference/training.md)
- [Experiment Design and Tracking](../../experiments/reference/experiments.md)

## Sources

- https://docs.pytorch.org/docs/2.15/distributed.fsdp.fully_shard.html — FSDP2 `fully_shard` and the FSDP1 migration pointer (accessed 2026-11-22)
- https://docs.pytorch.org/docs/2.15/generated/torch.load.html — `weights_only=True` default and `add_safe_globals` (accessed 2026-11-22)
- https://pytorch.org/blog/pytorch2-6/ — PyTorch 2.6 flips the `torch.load` default to `weights_only=True` (accessed 2026-11-22)
- https://docs.pytorch.org/docs/2.15/onnx.html — `torch.onnx.export(..., dynamo=True)` default (accessed 2026-11-22; TorchScript deprecation warning probed on torch 2.15.0+cpu)
- https://docs.pytorch.org/docs/2.15/amp.html — autocast dtypes and `GradScaler` (accessed 2026-11-22)
