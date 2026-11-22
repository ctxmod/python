# Experiments

Use Hydra for configuration management and MLflow for experiment tracking.

Use Hydra for hierarchical configuration management. Track all experiments with MLflow, logging parameters, metrics, and artifacts. Structure experiment directories with separate locations for configurations, checkpoints, and logs.

## Rationale

Hydra's composable YAML configurations enable reproducible experiments through configuration versioning and override syntax, eliminating hardcoded hyperparameters scattered across training scripts. MLflow provides a centralized experiment registry that tracks parameter-metric relationships across runs, making it possible to compare experiments and reproduce results. Standardized directory structure prevents checkpoint and log collisions between concurrent experiments.

[Experiment Design and Tracking](../reference/experiments.md)

## Examples

**Correct**:
```python
import hydra
import mlflow
from omegaconf import DictConfig

@hydra.main(config_path="configs", config_name="train", version_base=None)
def train(cfg: DictConfig) -> float:
    mlflow.set_experiment(cfg.experiment.name)
    with mlflow.start_run():
        mlflow.log_params(dict(cfg.model))
        val_loss = run_training(cfg)
        mlflow.log_metric("val_loss", val_loss)
    return val_loss
```

**Incorrect**:
```python
lr = 0.001  # Hardcoded hyperparameters, no tracking
model = Classifier(768, 10)
train(model, lr, 50)
```

## Exceptions

Quick debugging runs may bypass MLflow logging with a `--debug` flag that disables tracking. Single-experiment scripts under 200 lines may use dataclass-based configuration instead of Hydra when the overhead of YAML configuration files outweighs the benefit.

## Tooling

Install `hydra-core` and `mlflow` via uv. Run experiments with `python train.py` or override parameters with `python train.py training.lr=5e-4 model.dropout=0.2`.
