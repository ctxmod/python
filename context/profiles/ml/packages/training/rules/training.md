# Training

Training loops require checkpointing with model, optimizer, and RNG state for deterministic resumption.

Training loops must save checkpoints that include model state, optimizer state, learning rate scheduler state, and RNG states for deterministic resumption. Implement early stopping based on validation metrics. Report progress through structured logging at configurable intervals.

[Training, Checkpointing, and Resumption](../reference/training.md)
[Testing Philosophy](../../../../core/packages/testing/reference/testing.md)

## Examples

**Correct**:
```python
import torch
from pathlib import Path

def save_checkpoint(
    model: nn.Module,
    optimizer: torch.optim.Optimizer,
    scheduler: torch.optim.lr_scheduler.LRScheduler,
    epoch: int,
    val_loss: float,
    path: Path,
) -> None:
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "scheduler_state_dict": scheduler.state_dict(),
        "val_loss": val_loss,
        "rng_state": torch.get_rng_state(),
        "cuda_rng_state": torch.cuda.get_rng_state_all() if torch.cuda.is_available() else None,
    }, path)
```

**Incorrect**:
```python
# Missing optimizer and RNG state -- resumed training will diverge
torch.save(model.state_dict(), "model.pth")
```

## Exceptions

Inference-only scripts do not require checkpointing. Fine-tuning runs shorter than 30 minutes may checkpoint only at completion rather than at intervals, provided the time-to-retrain cost is acceptable.

## Tooling

Use `torch.save` and `torch.load` for checkpoint serialization. Configure checkpoint intervals through Hydra configuration. Monitor training progress with MLflow metric logging and tqdm progress bars.
