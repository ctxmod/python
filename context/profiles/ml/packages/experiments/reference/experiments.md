# Experiment Design and Tracking

Experiment design and tracking principles covering Hydra configuration, MLflow tracking, reproducibility, and directory organization.

ML experiments require three coordinated systems: configuration management for reproducible parameter sets, experiment tracking for parameter-metric correlation, and directory organization for artifact isolation. Hydra handles configuration, MLflow handles tracking, and a standardized directory layout prevents artifact collisions between concurrent runs.

## Configuration Management with Hydra

Hydra provides hierarchical YAML configuration with composition, override syntax, and multirun support. Store configurations in a `configs/` directory organized by concern (model, training, data).

### Configuration Composition

The root configuration composes from sub-configurations using Hydra's defaults list:

```yaml
# configs/train.yaml
defaults:
  - model: transformer
  - training: default
  - data: standard

experiment:
  name: "baseline"
  seed: 42

paths:
  output_dir: "outputs/${experiment.name}"
  checkpoint_dir: "${paths.output_dir}/checkpoints"
```

Override parameters from the command line without modifying YAML files:

```bash
python train.py model=cnn training.lr=5e-4 experiment.name="cnn-sweep-01"
```

### Configuration Guidelines

| Guideline | Rationale |
|-----------|-----------|
| Group by concern (model, training, data) | Enables independent variation of each axis |
| Use variable interpolation for paths | Derived paths update automatically |
| Version control all YAML configs | Configurations are part of the experiment record |
| Avoid Python-side defaults for hyperparameters | Hydra YAML is the single source of truth |
| Use `version_base=None` in `@hydra.main` | Explicit behavior, avoids deprecation warnings |

For hyperparameter sweeps, use Hydra's multirun mode (`--multirun`) or the Optuna sweeper plugin for Bayesian optimization.

## Experiment Tracking with MLflow

MLflow records parameters, metrics, and artifacts for each run. Structure tracking around three operations: logging parameters at run start, logging metrics during training, and logging artifacts at completion.

```python
import mlflow

def train_with_tracking(cfg: DictConfig) -> float:
    mlflow.set_tracking_uri(cfg.experiment.tracking_uri)
    mlflow.set_experiment(cfg.experiment.name)

    with mlflow.start_run():
        mlflow.log_params(_flatten_config(cfg))
        for epoch in range(cfg.training.epochs):
            train_loss = train_one_epoch(model, train_loader, optimizer)
            val_loss, metrics = evaluate(model, val_loader)
            mlflow.log_metrics({"train_loss": train_loss, "val_loss": val_loss, **metrics}, step=epoch)
        mlflow.log_artifact(str(best_checkpoint_path))
    return val_loss
```

### Tracking Guidelines

| What to Track | When | Method |
|--------------|------|--------|
| All hyperparameters | Run start | `mlflow.log_params()` |
| Training/validation loss | Each epoch | `mlflow.log_metrics(step=epoch)` |
| Best model checkpoint | Training end | `mlflow.log_artifact()` |
| Configuration file | Run start | `mlflow.log_artifact()` |

Use MLflow Model Registry to version models through lifecycle stages (Staging, Production). This provides an audit trail from training run to deployed model and enables rollback.

## Reproducibility

Reproducible experiments require controlling four sources of randomness: Python's random module, NumPy, PyTorch CPU, and PyTorch CUDA.

```python
import random, numpy as np, torch

def set_seed(seed: int) -> None:
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    if torch.cuda.is_available():
        torch.cuda.manual_seed_all(seed)
```

Full determinism with `torch.use_deterministic_algorithms(True)` disables non-deterministic CUDA operations. Enable for reproducibility-critical experiments; disable for production training where throughput matters more.

### Reproducibility Checklist

| Source | Control Method |
|--------|---------------|
| Python random | `random.seed(seed)` |
| NumPy random | `np.random.seed(seed)` |
| PyTorch CPU | `torch.manual_seed(seed)` |
| PyTorch CUDA | `torch.cuda.manual_seed_all(seed)` |
| cuDNN algorithms | `torch.backends.cudnn.benchmark = False` |
| Data loading order | Seeded `Sampler` or `Generator` |

## Directory Organization

Standardize experiment output structure to prevent artifact collisions:

```
outputs/
  experiment-name/
    run-2026-02-21/
      checkpoints/
      logs/
      artifacts/
```

Hydra automatically creates timestamped output directories. Combine with MLflow artifact logging for both local access and centralized tracking.

## Evidence

Hydra configuration management follows Meta AI's internal ML workflow patterns and has been adopted by major research projects (NeMo, Detectron2). MLflow tracking is used across industry for experiment management. Seed-based reproducibility follows PyTorch's official documentation.

## Related

- [PyTorch Patterns](../../framework/reference/pytorch.md)
- [Training, Checkpointing, and Resumption](../../training/reference/training.md)
- [Project Structure](../../../../core/packages/structure/reference/structure.md)
