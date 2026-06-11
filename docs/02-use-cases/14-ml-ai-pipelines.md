---
layout: default
title: "Cadence for ML & AI Pipelines: Model Training, ETL, and Agent Orchestration"
description: How data and ML teams use Cadence to orchestrate model training pipelines, ETL workflows, feature engineering, and AI agent task graphs with durable execution, GPU routing, and automatic retries.
keywords:
  - cadence machine learning workflow
  - cadence ML pipeline
  - cadence AI orchestration
  - cadence model training workflow
  - workflow engine for ML
  - cadence ETL pipeline
  - cadence data pipeline
  - durable execution ML
  - cadence AI agent
  - cadence feature engineering
permalink: /docs/use-cases/ml-ai-pipelines
---

# ML & AI Pipelines: Model Training, ETL, and Agent Orchestration

ML and data pipelines are among the most workflow-heavy workloads in engineering: long-running training jobs, multi-stage ETL transforms, feature engineering chains, model evaluation, and deployment gates. Failures mid-pipeline are expensive — a crashed training job that restarts from scratch wastes GPU hours and delays model rollouts.

Cadence's durable execution model makes it a natural fit for ML infrastructure: pipelines resume from exactly where they failed, GPU-bound steps route to the right hosts, and every run has a full history for debugging and reproducibility.

## The problem with ad-hoc ML pipelines

A common ML pipeline architecture stitches together shell scripts, cron jobs, Airflow DAGs, and Kubernetes jobs. Each layer adds operational overhead: another scheduler to maintain, another failure mode to handle, another place to look when a job silently disappears.

Cadence replaces this stack with plain application code. Pipeline steps are activities — ordinary functions with retry policies and timeouts. The workflow engine tracks state, handles failures, and provides visibility into every step without additional infrastructure.

## Model training pipelines

A model training workflow orchestrates data prep, training, evaluation, and registration as a sequence of durable steps:

```go
func ModelTrainingWorkflow(ctx workflow.Context, config TrainingConfig) error {
    ao := workflow.ActivityOptions{
        StartToCloseTimeout: 4 * time.Hour, // training can take hours
        RetryPolicy: &cadence.RetryPolicy{
            MaximumAttempts: 3,
        },
    }
    ctx = workflow.WithActivityOptions(ctx, ao)

    var dataPath string
    if err := workflow.ExecuteActivity(ctx, PrepareTrainingData, config).Get(ctx, &dataPath); err != nil {
        return err
    }

    var modelPath string
    if err := workflow.ExecuteActivity(ctx, TrainModel, dataPath, config).Get(ctx, &modelPath); err != nil {
        return err
    }

    var metrics EvalMetrics
    if err := workflow.ExecuteActivity(ctx, EvaluateModel, modelPath).Get(ctx, &metrics); err != nil {
        return err
    }

    if metrics.Accuracy < config.MinAccuracy {
        return fmt.Errorf("model accuracy %.2f below threshold %.2f", metrics.Accuracy, config.MinAccuracy)
    }

    return workflow.ExecuteActivity(ctx, RegisterModel, modelPath, metrics).Get(ctx, nil)
}
```

If the training activity crashes mid-run, Cadence resumes the workflow — but the training activity restarts from scratch. For very long training jobs, use checkpointing within the activity or split training into sub-activities with intermediate checkpoints.

## GPU and host routing

One of Cadence's most useful features for ML is **task routing** — the ability to direct an activity to a specific worker or class of workers. This is critical when:

- Models are sharded by entity (city, user segment, product category) and each shard is pre-loaded on specific hosts
- GPU workers are a limited resource and need to be targeted explicitly
- Different pipeline stages require different hardware profiles (CPU-heavy data prep vs. GPU-heavy training)

```go
// Route the inference activity to workers with model shard for "city:nyc"
gpuCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
    TaskList:            "gpu-workers-shard-nyc",
    StartToCloseTimeout: 30 * time.Minute,
})
workflow.ExecuteActivity(gpuCtx, RunInference, request)
```

## ETL pipelines

Cadence handles large-scale ETL by combining:

- **Partitioned scan**: split input data into chunks and process in parallel using child workflows or activity fan-out
- **Durable progress**: each partition checkpoints independently — a failure in one partition doesn't restart the entire job
- **Rate limiting**: Cadence's global rate limiters prevent ETL fan-out from overwhelming downstream systems

See the [Partitioned Scan use case](/docs/use-cases/partitioned-scan) for a detailed fan-out pattern.

## AI agent orchestration

Cadence is well-suited for orchestrating multi-step AI agent task graphs — sequences of LLM calls, tool invocations, and retrieval steps that must complete reliably even when individual calls fail or time out.

Each agent step is an activity:
- **Retries**: transient LLM API errors (rate limits, timeouts) are retried automatically
- **Long waits**: `workflow.GetSignalChannel` pauses the agent to wait for human feedback or an external event
- **Branching**: workflow code handles conditional logic (e.g. if confidence < threshold, escalate to human review)
- **Parallelism**: multiple tool calls execute concurrently with `workflow.Go`

```go
func AgentWorkflow(ctx workflow.Context, task AgentTask) (string, error) {
    var plan []Step
    if err := workflow.ExecuteActivity(ctx, PlanTask, task).Get(ctx, &plan); err != nil {
        return "", err
    }

    // Execute tool calls in parallel
    futures := make([]workflow.Future, len(plan))
    for i, step := range plan {
        futures[i] = workflow.ExecuteActivity(ctx, ExecuteTool, step)
    }

    results := make([]ToolResult, len(plan))
    for i, f := range futures {
        if err := f.Get(ctx, &results[i]); err != nil {
            return "", err
        }
    }

    var answer string
    err := workflow.ExecuteActivity(ctx, SynthesizeAnswer, task, results).Get(ctx, &answer)
    return answer, err
}
```

## Feature engineering workflows

Feature pipelines are a natural fit for Cadence:

- **Periodic re-computation**: use a cron workflow to recompute features on a schedule (hourly, daily)
- **Event-driven updates**: trigger feature recomputation on upstream data changes via signals
- **Dependency chains**: model feature dependencies as workflow steps — downstream features wait for upstream activities to complete

## Key properties for ML & data teams

| Requirement | How Cadence handles it |
|---|---|
| **Fault-tolerant pipelines** | Workflow resumes from the failed step — not from scratch |
| **GPU / host routing** | Task list routing directs activities to specific worker pools |
| **Long-running jobs** | Activities run for hours or days with no polling or heartbeat hacks |
| **Pipeline visibility** | Full step history in the Cadence Web UI — inputs, outputs, retries |
| **Parallel fan-out** | Child workflows and `workflow.Go` parallelize across partitions |
| **Rate limiting** | Global rate limiters prevent fan-out from overwhelming downstream APIs |
| **Reproducibility** | Immutable workflow history records every input and output |

## Getting started

- [Cadence Go SDK quickstart](/docs/get-started/)
- [Activity options: timeouts and retries](/docs/go-client/activities)
- [Child workflows: fan-out patterns](/docs/go-client/child-workflows)
- [Task routing: targeting specific workers](/docs/go-client/workers)
