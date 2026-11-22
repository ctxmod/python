# Training Workflow

Training workflow covering training loop implementation, checkpointing, resumption, and progress monitoring.

Procedure for implementing a training loop with checkpointing, early stopping, and progress monitoring. The result is a training script that can be interrupted and resumed without losing progress.

## Prerequisites

- Experiment environment set up (Hydra, MLflow, directory structure)
- Model defined as `nn.Module` with typed `forward` method
- Dataset classes implemented with `__getitem__` and `__len__`

- [Experiment Setup](../../experiments/reference/experiment_setup.md)

## Training Workflow

### 1. Initialize Model and Data

Create model, optimizer, scheduler, and data loaders from configuration:

```python
def setup_training(cfg: DictConfig, device: torch.device) -> TrainingComponents:
    model = build_model(cfg.model).to(device)
    optimizer = torch.optim.AdamW(model.parameters(), lr=cfg.training.lr, weight_decay=cfg.training.weight_decay)
    scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, mode="min", patience=3)
    train_loader = DataLoader(
        train_dataset, batch_size=cfg.training.batch_size,
        shuffle=True, num_workers=4, pin_memory=True, persistent_workers=True,
    )
    val_loader = DataLoader(val_dataset, batch_size=cfg.training.batch_size * 2, shuffle=False, num_workers=4)
    return TrainingComponents(model, optimizer, scheduler, nn.CrossEntropyLoss(), train_loader, val_loader)
```

- [PyTorch Patterns](../../framework/reference/pytorch.md)

### 2. Implement Checkpoint Resumption

Check for existing checkpoints before training starts:

```python
def maybe_resume(
    checkpoint_dir: Path, model: nn.Module,
    optimizer: torch.optim.Optimizer, scheduler: torch.optim.lr_scheduler.LRScheduler,
) -> int:
    checkpoints = sorted(checkpoint_dir.glob("epoch_*.pt"))
    if not checkpoints:
        return 0
    checkpoint = torch.load(checkpoints[-1], weights_only=False)
    model.load_state_dict(checkpoint["model_state_dict"])
    optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
    scheduler.load_state_dict(checkpoint["scheduler_state_dict"])
    torch.set_rng_state(checkpoint["rng_state"])
    if checkpoint.get("cuda_rng_state") is not None:
        torch.cuda.set_rng_state_all(checkpoint["cuda_rng_state"])
    return checkpoint["epoch"] + 1
```

### 3. Implement Training and Validation Epochs

```python
def train_one_epoch(
    model: nn.Module, loader: DataLoader, optimizer: torch.optim.Optimizer,
    criterion: nn.Module, device: torch.device, grad_clip: float = 1.0,
) -> float:
    model.train()
    total_loss = 0.0
    for batch in tqdm(loader, desc="Training", leave=False):
        optimizer.zero_grad()
        loss = criterion(model(batch.inputs.to(device)), batch.labels.to(device))
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=grad_clip)
        optimizer.step()
        total_loss += loss.item()
    return total_loss / len(loader)

@torch.no_grad()
def evaluate(model: nn.Module, loader: DataLoader, criterion: nn.Module, device: torch.device) -> tuple[float, dict[str, float]]:
    model.eval()
    total_loss = 0.0
    all_preds, all_labels = [], []
    for batch in loader:
        outputs = model(batch.inputs.to(device))
        total_loss += criterion(outputs, batch.labels.to(device)).item()
        all_preds.append(outputs.argmax(dim=-1))
        all_labels.append(batch.labels.to(device))
    preds, labels = torch.cat(all_preds), torch.cat(all_labels)
    return total_loss / len(loader), {"accuracy": (preds == labels).float().mean().item()}
```

### 4. Assemble the Training Loop

```python
def train(model, train_loader, val_loader, optimizer, scheduler, criterion, config, device):
    best_val_loss = float("inf")
    early_stopping = EarlyStopping(patience=config.patience)
    for epoch in range(config.start_epoch, config.epochs):
        train_loss = train_one_epoch(model, train_loader, optimizer, criterion, device)
        val_loss, metrics = evaluate(model, val_loader, criterion, device)
        scheduler.step(val_loss)
        logger.info("Epoch %d/%d - train: %.4f, val: %.4f", epoch + 1, config.epochs, train_loss, val_loss)
        if epoch % config.checkpoint_interval == 0:
            save_checkpoint(model, optimizer, scheduler, epoch, val_loss, config.checkpoint_dir / f"epoch_{epoch}.pt")
            cleanup_old_checkpoints(config.checkpoint_dir, keep=3)
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            save_checkpoint(model, optimizer, scheduler, epoch, val_loss, config.checkpoint_dir / "best.pt")
        if early_stopping.should_stop(val_loss):
            logger.info("Early stopping at epoch %d", epoch + 1)
            break
    return {"best_val_loss": best_val_loss, **metrics}
```

- [Training, Checkpointing, and Resumption](training.md)

### 5. Verify the Training Script

```bash
just train training.epochs=3 training.batch_size=8 experiment.name="verify"
```

Verify: loss decreases, checkpoints appear, MLflow logs metrics. Test resumption by interrupting and restarting.

## Decision Support

### Optimizer Selection

| Optimizer | Learning Rate | Use Case |
|-----------|--------------|----------|
| AdamW | 1e-3 to 1e-4 | Default for most tasks |
| SGD + Momentum | 1e-1 to 1e-2 | CNNs with tuned schedule |
| LAMB | 1e-2 to 1e-3 | Large-batch distributed |

## Troubleshooting

**Loss is NaN**: Reduce learning rate by 10x, add gradient clipping, check for division by zero.

**Validation loss increases from epoch 1**: Learning rate too high or data loading error.

**Training slower than expected**: Check `num_workers`, `pin_memory=True`, and profile with `torch.profiler`.

## Related

- [Experiment Design and Tracking](../../experiments/reference/experiments.md)
