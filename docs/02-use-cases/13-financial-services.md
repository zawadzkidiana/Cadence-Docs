---
layout: default
title: "Cadence for Financial Services: Payment Orchestration & Fintech Workflows"
description: How financial services teams use Cadence for payment orchestration, loan origination, fraud detection pipelines, and multi-step financial transaction workflows with guaranteed completion and full audit trails.
keywords:
  - cadence financial services
  - cadence payment orchestration
  - cadence fintech workflow
  - cadence payment workflow
  - workflow engine for payments
  - cadence loan origination
  - cadence fraud detection workflow
  - durable execution fintech
  - cadence banking workflow
  - payment orchestration workflow engine
permalink: /docs/use-cases/financial-services
---

# Financial Services: Payment Orchestration & Fintech Workflows

Financial services teams operate some of the most demanding workflow requirements: transactions must complete exactly once, compensation logic must run when a step fails, audit trails must be complete, and the system must stay operational at all times. Cadence is a proven fit for these requirements — [DoorDash](https://doordash.engineering/2022/05/18/enabling-faster-financial-partnership-integrations-using-cadence/) and Uber both run production financial workflows on Cadence at scale.

## Why financial workflows are hard without durable execution

A typical payment or financial transaction involves multiple steps: validate the request, check fraud signals, debit an account, call a downstream payment provider, confirm settlement, send a notification, update a ledger. Each step can fail. Steps may need to run in a specific order. Some steps must be rolled back if a later step fails (the Saga pattern).

Without a durable execution engine, teams reach for queues, cron jobs, and database polling to coordinate these steps. The result is fragile: failures leave the system in an unknown state, retries cause double-charges, and getting the current status of any individual transaction is nearly impossible.

Cadence replaces that infrastructure with plain application code. The workflow engine guarantees that your code runs to completion — survives crashes, restarts, and downstream outages — and gives you full visibility into every step of every transaction.

## Payment orchestration

A payment orchestration workflow coordinates authorization, settlement, and notification as a sequence of durable activities. If the payment provider is temporarily unavailable, Cadence retries automatically with configurable backoff. If settlement fails after authorization, compensation logic rolls back the hold.

```go
func PaymentWorkflow(ctx workflow.Context, req PaymentRequest) error {
    ao := workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
        RetryPolicy: &cadence.RetryPolicy{
            MaximumAttempts: 5,
            InitialInterval: time.Second,
        },
    }
    ctx = workflow.WithActivityOptions(ctx, ao)

    // Each step is retried automatically on failure
    if err := workflow.ExecuteActivity(ctx, ValidateFunds, req).Get(ctx, nil); err != nil {
        return err
    }
    if err := workflow.ExecuteActivity(ctx, AuthorizePayment, req).Get(ctx, nil); err != nil {
        return err
    }
    if err := workflow.ExecuteActivity(ctx, SettlePayment, req).Get(ctx, nil); err != nil {
        // Compensation: reverse the authorization
        _ = workflow.ExecuteActivity(ctx, ReverseAuthorization, req).Get(ctx, nil)
        return err
    }
    return workflow.ExecuteActivity(ctx, SendConfirmation, req).Get(ctx, nil)
}
```

Because workflow state is persisted at every step, a crash between authorization and settlement does not leave the payment in an unknown state — Cadence resumes from exactly where it stopped.

## Loan origination

Loan origination involves steps across multiple systems: credit check, document collection, underwriting, compliance review, disbursement. Some steps wait for human review. Some steps can run in parallel. All steps need a complete audit trail.

Cadence handles this naturally:

- **Long-running waits**: `workflow.GetSignalChannel` waits for a human approval signal without blocking a thread.
- **Parallel steps**: `workflow.Go` runs credit checks and document verification concurrently.
- **Audit trail**: Cadence's workflow history is an immutable record of every step, input, output, and timestamp — ready for compliance or dispute resolution.
- **Timeouts**: `workflow.NewTimer` automatically escalates if a review hasn't completed within SLA.

## Fraud detection pipelines

Fraud detection often requires coordinating signals from multiple systems — transaction history, device fingerprinting, behavioral patterns — and acting within milliseconds. Cadence supports this through:

- **Activity routing**: route fraud-scoring activities to specific hosts that hold pre-loaded models, avoiding cold-start latency.
- **Parallel signal gathering**: gather signals from multiple fraud systems concurrently and combine results.
- **Adaptive response**: use workflow signals to inject real-time risk updates from external systems mid-execution.

## Saga pattern: coordinated rollback

The [Saga pattern](https://microservices.io/patterns/data/saga.html) is a standard approach for distributed transactions that require compensation on failure. Cadence makes Saga straightforward: each forward step is an activity, and each compensation step is triggered in a deferred function or explicit error branch.

DoorDash uses this pattern to coordinate financial partnership integrations — see their write-up: [Enabling Faster Financial Partnership Integrations Using Cadence](https://doordash.engineering/2022/05/18/enabling-faster-financial-partnership-integrations-using-cadence/).

## Key properties for financial services

| Requirement | How Cadence handles it |
|---|---|
| **Exactly-once execution** | Workflow state is persisted at every step — no double-execution on retry |
| **Compensation on failure** | Explicit rollback activities in error branches (Saga) |
| **Audit trail** | Immutable workflow history: every step, input, output, and timestamp |
| **Long-running transactions** | Workflows run for days, months, or years without polling |
| **Human-in-the-loop** | `workflow.GetSignalChannel` waits for external approvals without blocking |
| **High throughput** | Millions of concurrent payment workflows per Cadence environment |
| **Observability** | Full workflow state visible in the Cadence Web UI at any point |

## Getting started

- [Cadence Go SDK quickstart](/docs/get-started/)
- [Activity retry policies](/docs/go-client/retries)
- [Signals: waiting for external input](/docs/go-client/signals)
- [Cadence at DoorDash: financial partnership integrations](https://doordash.engineering/2022/05/18/enabling-faster-financial-partnership-integrations-using-cadence/)
