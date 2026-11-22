# Experiment Setup

Experiment setup workflow covering Hydra configuration, MLflow tracking, directory structure, and verification.

Procedure for setting up ML experiments with Hydra configuration, MLflow tracking, and standardized directory structure. The result is a reproducible experiment environment ready for training.

## Prerequisites

Gather before starting:

- **Experiment purpose**: Hypothesis or model being trained.
- **Model architecture**: Module structure and hyperparameter ranges.
- **Dataset specification**: Training, validation, and test splits.
- **Compute requirements**: GPU type, memory, expected duration.
- **Base profile setup**: Python project initialized with uv, basedpyright, ruff, pytest.

## Setup Workflow

### 1. Install ML Dependencies

```bash
uv add torch torchvision hydra-core omegaconf mlflow
uv add --dev pytest-benchmark
```

For CUDA support, configure the PyTorch index in `pyproject.toml`:

```toml
[[tool.uv.index]]
name = "pytorch-cuda"
url = "https://download.pytorch.org/whl/cu124"
```

### 2. Create Configuration Structure

```
configs/
  train.yaml            # Root configuration
  model/
    default.yaml         # Model hyperparameters
  training/
    default.yaml         # Training hyperparameters
  data/
    default.yaml         # Data processing settings
```

```yaml
# configs/train.yaml
defaults:
  - model: default
  - training: default
  - data: default

experiment:
  name: "experiment-name"
  seed: 42
  tracking_uri: "mlruns/"

paths:
  output_dir: "outputs/${experiment.name}"
  checkpoint_dir: "${paths.output_dir}/checkpoints"
```

[Experiment Design and Tracking](experiments.md)

### 3. Create the Training Entry Point

```python
import hydra
import mlflow
from omegaconf import DictConfig, OmegaConf

@hydra.main(config_path="configs", config_name="train", version_base=None)
def main(cfg: DictConfig) -> float:
    set_seed(cfg.experiment.seed)
    device = get_device()
    model = build_model(cfg.model).to(device)
    optimizer = torch.optim.AdamW(model.parameters(), lr=cfg.training.lr)

    mlflow.set_tracking_uri(cfg.experiment.tracking_uri)
    mlflow.set_experiment(cfg.experiment.name)
    with mlflow.start_run():
        mlflow.log_params(OmegaConf.to_container(cfg, resolve=True))
        result = train(model, train_loader, val_loader, optimizer, scheduler, criterion, cfg.training, device)
        mlflow.log_metrics(result)
    return result["best_val_loss"]
```

[PyTorch Patterns](../../framework/reference/pytorch.md)

### 4. Set Up Directory Structure

```
project/
  src/project_name/
    models/          # nn.Module definitions
    data/            # Dataset classes and transforms
    training/        # Training loop and utilities
    evaluation/      # Metrics and evaluation
  configs/           # Hydra YAML configurations
  outputs/           # Experiment outputs (gitignored)
  tests/
```

Add to `.gitignore`: `outputs/`, `mlruns/`, `checkpoints/`, `*.pt`, `*.pth`.

### 5. Add Justfile Tasks

Add `train`, `sweep`, and `mlflow-ui` tasks to the justfile for common operations.

### 6. Verify the Setup

```bash
just train training.epochs=1 experiment.name="smoke-test"
uv run basedpyright src/
uv run pytest tests/ -v
```

Verify: Hydra resolves configuration, MLflow creates a run with logged parameters, a checkpoint appears in the output directory, and the training loop completes one epoch.

[Model Evaluation](../../evaluation/reference/model_evaluation.md)

## Decision Support

### Tracking Backend Selection

| Backend | Use Case | Setup Complexity |
|---------|----------|-----------------|
| MLflow (local) | Solo development | Low |
| MLflow (server) | Team collaboration | Medium |
| Weights & Biases | Cloud-native teams | Low |
| TensorBoard | Quick visualization | Low |

## Troubleshooting

**Hydra cannot find config**: Verify `config_path` is relative to the script location.

**CUDA out of memory**: Reduce `batch_size`. Consider gradient accumulation for larger effective batch sizes.

**Non-deterministic results**: Verify all four random seeds are set. Enable `torch.use_deterministic_algorithms(True)` for full determinism.

## Related

- [Training, Checkpointing, and Resumption](../../training/reference/training.md)
- [Project Setup](../../../../core/packages/toolchain/reference/project_setup.md)
