# Model Evaluation

Model evaluation workflow covering checkpoint loading, metric computation, baseline comparison, and inference benchmarking.

Procedure for evaluating trained models against test datasets, comparing to baselines, and benchmarking inference performance. Run after training completes and before deploying to production.

## Prerequisites

- Trained model with best checkpoint saved
- Test dataset separate from training and validation splits
- Baseline metrics for comparison (previous model or published benchmark)
- Target inference latency (if serving)

## Evaluation Workflow

### 1. Load the Best Checkpoint

```python
def load_model_for_eval(checkpoint_path: Path, model: nn.Module, device: torch.device) -> nn.Module:
    checkpoint = torch.load(checkpoint_path, map_location=device, weights_only=False)
    model.load_state_dict(checkpoint["model_state_dict"])
    model.eval()
    return model
```

[Training, Checkpointing, and Resumption](../../training/reference/training.md)

### 2. Compute Evaluation Metrics

Run inference on the test set and compute task-specific metrics:

```python
@torch.no_grad()
def evaluate_model(model: nn.Module, test_loader: DataLoader, criterion: nn.Module, device: torch.device) -> EvaluationResult:
    model.eval()
    all_preds, all_labels = [], []
    total_loss = 0.0
    for batch in test_loader:
        outputs = model(batch.inputs.to(device))
        total_loss += criterion(outputs, batch.labels.to(device)).item()
        all_preds.append(outputs.argmax(dim=-1).cpu())
        all_labels.append(batch.labels.cpu())
    return compute_metrics(total_loss / len(test_loader), torch.cat(all_preds), torch.cat(all_labels))
```

### Metric Selection by Task

| Task | Primary Metrics | Secondary Metrics |
|------|----------------|-------------------|
| Classification | Accuracy, F1 (macro) | Precision, Recall, AUC-ROC |
| Regression | MSE, MAE | R-squared, MAPE |
| Ranking | NDCG, MRR | MAP, Hit Rate |
| Generation | BLEU, ROUGE | Perplexity, BERTScore |

Log all metrics to MLflow for cross-run comparison.

[Experiment Design and Tracking](../../experiments/reference/experiments.md)

### 3. Compare Against Baselines

| Scenario | Decision |
|----------|----------|
| All metrics improve | Promote to production candidate |
| Primary improves, secondary mixed | Review trade-offs, likely promote |
| Primary unchanged, secondary improve | Promote if secondary is business-critical |
| Any primary metric regresses | Investigate, do not promote |

### 4. Benchmark Inference Performance

```python
def benchmark_inference(model: nn.Module, sample_input: Tensor, device: torch.device, num_runs: int = 100) -> dict[str, float]:
    model.eval()
    sample_input = sample_input.to(device)
    for _ in range(10):  # Warmup
        with torch.no_grad():
            model(sample_input)
    if device.type == "cuda":
        torch.cuda.synchronize()
    latencies = []
    for _ in range(num_runs):
        start = time.perf_counter()
        with torch.no_grad():
            model(sample_input)
        if device.type == "cuda":
            torch.cuda.synchronize()
        latencies.append(time.perf_counter() - start)
    return {
        "mean_latency_ms": sum(latencies) / len(latencies) * 1000,
        "p95_latency_ms": sorted(latencies)[int(len(latencies) * 0.95)] * 1000,
        "throughput_per_sec": len(latencies) / sum(latencies),
    }
```

### 5. Check Model Size

```python
def model_size_info(model: nn.Module) -> dict[str, float]:
    param_count = sum(p.numel() for p in model.parameters())
    size_mb = sum(p.nelement() * p.element_size() for p in model.parameters()) / (1024 * 1024)
    return {"total_parameters": param_count, "model_size_mb": round(size_mb, 2)}
```

### 6. Run and Review

```bash
uv run python evaluate.py checkpoint=outputs/experiment/checkpoints/best.pt
```

[PyTorch Patterns](../../framework/reference/pytorch.md)

## Thresholds

| Metric | Pass | Warning | Fail |
|--------|------|---------|------|
| Test accuracy | Meets target | Within 2% of target | >2% below target |
| Baseline comparison | Improves primary | Matches baseline | Regresses |
| Inference latency (p95) | <100ms | 100-200ms | >200ms |
| Model size | <500MB | 500MB-1GB | >1GB |
| Metric consistency | Cross-val std <2% | 2-5% std | >5% std |

Adjust thresholds based on task and deployment target. Real-time serving requires stricter latency bounds. Batch inference tolerates larger models.

## Troubleshooting

**Evaluation metrics differ from validation**: Check for data leakage. Verify test set has no overlap with training data. Ensure identical preprocessing.

**Inference latency higher than expected**: Profile with `torch.profiler`. Try `torch.compile(model)` for PyTorch 2.0+. Consider quantization.

**Benchmark results vary**: Increase `num_runs`. Ensure no other GPU workloads. Use `torch.cuda.synchronize()` for accurate timing.

## Related

- [Training Workflow](../../training/reference/training_workflow.md)
