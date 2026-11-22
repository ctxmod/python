# Training, Checkpointing, and Resumption

Training loop principles covering checkpointing, early stopping, learning rate scheduling, gradient management, and resumption.

Training loops coordinate five activities: forward-backward computation, checkpointing for fault tolerance, learning rate scheduling for convergence, gradient management for stability, and progress reporting for observability.

## Checkpointing

Save complete training state for deterministic resumption. A checkpoint must contain everything required to continue training as if it had never stopped.

### Required Checkpoint Contents

| Component | Purpose | Missing Impact |
|-----------|---------|---------------|
| `model.state_dict()` | Model weights | Cannot resume |
| `optimizer.state_dict()` | Momentum, adaptive rates | Training diverges |
| `scheduler.state_dict()` | LR schedule position | LR resets to initial |
| `epoch` | Progress counter | Repeats completed epochs |
| `torch.get_rng_state()` | CPU random state | Non-deterministic data ordering |
| `torch.cuda.get_rng_state_all()` | GPU random state | Non-deterministic dropout |

```python
def save_checkpoint(
    model: nn.Module, optimizer: torch.optim.Optimizer,
    scheduler: torch.optim.lr_scheduler.LRScheduler,
    epoch: int, val_loss: float, path: Path,
) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
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

### Checkpoint Frequency

| Strategy | Frequency | Use Case |
|----------|-----------|----------|
| Fixed interval | Every N epochs | Predictable storage |
| Best-only | On validation improvement | Minimal storage |
| Hybrid | Interval + best | Balance of safety and storage |

Keep 3-5 recent interval checkpoints and always retain the best checkpoint.

## Early Stopping

Halt training when validation metrics stop improving. Set `patience` to 5-10 epochs for standard training, 10-20 for fine-tuning where convergence is slower. The `min_delta` parameter prevents false triggers from noise.

```python
class EarlyStopping:
    def __init__(self, patience: int, min_delta: float = 0.0) -> None:
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_loss: float | None = None

    def should_stop(self, val_loss: float) -> bool:
        if self.best_loss is None or val_loss < self.best_loss - self.min_delta:
            self.best_loss = val_loss
            self.counter = 0
            return False
        self.counter += 1
        return self.counter >= self.patience
```

## Learning Rate Scheduling

| Scheduler | Pattern | Use Case |
|-----------|---------|----------|
| `ReduceLROnPlateau` | Reduce on validation stall | General training, adaptive |
| `CosineAnnealingLR` | Cosine decay to minimum | Fixed-length training |
| `OneCycleLR` | Warm-up then decay | Fast convergence |
| `LinearLR` + `CosineAnnealingLR` | Linear warm-up, cosine decay | Transformer fine-tuning |

Warm-up prevents large early gradients from destabilizing training, particularly for transformers and large batch sizes.

## Gradient Management

Clip gradients to prevent explosion, especially with recurrent architectures or deep transformers:

```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

For limited GPU memory, use gradient accumulation to simulate larger batch sizes:

```python
accumulation_steps = 4
for i, batch in enumerate(train_loader):
    loss = criterion(model(batch.inputs), batch.labels) / accumulation_steps
    loss.backward()
    if (i + 1) % accumulation_steps == 0:
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()
        optimizer.zero_grad()
```

## Progress Reporting

Report training progress through multiple channels:

| Channel | Audience | Content |
|---------|----------|---------|
| Structured logging | Engineers | Loss, LR, epoch, timing |
| MLflow metrics | ML engineers | All metrics with step |
| tqdm progress bar | Interactive sessions | Batch progress within epoch |
| TensorBoard | Visual analysis | Loss curves, histograms |

Combine channels rather than choosing one. Each serves a distinct purpose: logs persist for debugging, MLflow enables comparison, tqdm provides interactive feedback.

## Rationale

ML training runs span hours to days, making fault tolerance through checkpointing essential. Saving optimizer state preserves momentum and adaptive learning rate statistics (Adam, AdamW), without which resumed training diverges from continuous runs. RNG state preservation ensures identical data ordering and dropout behavior across restarts, enabling reproducible results. Early stopping prevents overfitting by halting training when validation metrics plateau.

## Evidence

Checkpointing patterns follow PyTorch's official serialization documentation. Early stopping and LR scheduling strategies draw from established deep learning practices. Gradient accumulation enables effective large-batch training on consumer hardware.

## Related

- [PyTorch Patterns](../../framework/reference/pytorch.md)
- [Experiment Design and Tracking](../../experiments/reference/experiments.md)
- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
