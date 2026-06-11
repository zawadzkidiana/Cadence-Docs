---
layout: default
title: "Cadence for Microservices Orchestration: Multi-Step APIs & Saga Pattern"
description: How engineering teams use Cadence to orchestrate multi-step microservice workflows with guaranteed completion, automatic retries, compensation logic, and full visibility — replacing fragile queue-based coordination.
keywords:
  - cadence microservices orchestration
  - cadence service orchestration
  - microservices workflow engine
  - cadence saga pattern
  - cadence distributed transaction
  - cadence retry orchestration
  - workflow engine microservices
  - cadence compensation logic
  - cadence multi-step API
  - durable execution microservices
permalink: /docs/use-cases/microservices-orchestration
---

# Microservices Orchestration: Multi-Step APIs & Saga Pattern

Modern backend systems are collections of microservices. A single business operation — creating an order, onboarding a user, provisioning infrastructure — can require coordinating calls to 5, 10, or 20 downstream services. When any step can fail, the coordination code becomes the hardest part of the system.

The standard approach — queues, state machines in databases, and polling loops — works until it doesn't. Failures leave the system in ambiguous states. Getting the status of any individual operation requires querying multiple systems. Adding a new step means touching multiple services and hoping the retry logic is correct.

Cadence replaces this coordination layer with plain application code. Each step is an activity. The workflow engine guarantees completion, handles retries, and gives you full visibility into every operation.

## The coordination problem

Consider a user onboarding flow:

1. Create account in the identity service
2. Provision a default workspace
3. Send a welcome email
4. Enroll in the onboarding drip campaign
5. Notify the sales CRM

Without a workflow engine, this is typically implemented with a queue per step, a state machine in a database, and cron jobs to detect and retry stuck records. If step 3 fails, does the system retry step 3 only, or steps 3–5? If the service crashes between steps 2 and 3, does the state machine reflect that? How do you query which step a specific user is on?

With Cadence, this is a 30-line workflow function. State is tracked automatically. Each step retries independently. The Cadence Web UI shows exactly which step every user is on at any moment.

## A service orchestration workflow

```go
func OnboardingWorkflow(ctx workflow.Context, user User) error {
    ao := workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
        RetryPolicy: &cadence.RetryPolicy{
            MaximumAttempts:    10,
            InitialInterval:    time.Second,
            MaximumInterval:    5 * time.Minute,
            BackoffCoefficient: 2,
        },
    }
    ctx = workflow.WithActivityOptions(ctx, ao)

    if err := workflow.ExecuteActivity(ctx, CreateAccount, user).Get(ctx, nil); err != nil {
        return err
    }
    if err := workflow.ExecuteActivity(ctx, ProvisionWorkspace, user).Get(ctx, nil); err != nil {
        return err
    }

    // Run notification steps in parallel
    emailFuture := workflow.ExecuteActivity(ctx, SendWelcomeEmail, user)
    crmFuture := workflow.ExecuteActivity(ctx, NotifyCRM, user)
    enrollFuture := workflow.ExecuteActivity(ctx, EnrollDripCampaign, user)

    // Wait for all to complete
    for _, f := range []workflow.Future{emailFuture, crmFuture, enrollFuture} {
        if err := f.Get(ctx, nil); err != nil {
            return err
        }
    }
    return nil
}
```

Each step retries automatically on transient failures. Parallel steps execute concurrently. The workflow history records every step, its inputs, outputs, and retry count.

## Saga pattern: compensating transactions

When a multi-step operation must roll back on failure, the [Saga pattern](https://microservices.io/patterns/data/saga.html) is the standard approach. Cadence makes Saga straightforward: forward steps are activities, compensation steps are triggered in error branches.

```go
func OrderWorkflow(ctx workflow.Context, order Order) error {
    ao := workflow.ActivityOptions{StartToCloseTimeout: 30 * time.Second}
    ctx = workflow.WithActivityOptions(ctx, ao)

    if err := workflow.ExecuteActivity(ctx, ReserveInventory, order).Get(ctx, nil); err != nil {
        return err
    }

    if err := workflow.ExecuteActivity(ctx, ChargePayment, order).Get(ctx, nil); err != nil {
        // Compensate: release the inventory reservation
        _ = workflow.ExecuteActivity(ctx, ReleaseInventory, order).Get(ctx, nil)
        return err
    }

    if err := workflow.ExecuteActivity(ctx, ArrangeShipping, order).Get(ctx, nil); err != nil {
        // Compensate: refund payment and release inventory
        _ = workflow.ExecuteActivity(ctx, RefundPayment, order).Get(ctx, nil)
        _ = workflow.ExecuteActivity(ctx, ReleaseInventory, order).Get(ctx, nil)
        return err
    }

    return nil
}
```

Compensation activities are themselves retried — so a failed refund won't be silently lost.

## Long-running orchestration with human approval

Some orchestration workflows require human review steps. Cadence's signal mechanism pauses the workflow waiting for an external event — no polling, no database flag:

```go
func ProvisioningWorkflow(ctx workflow.Context, req ProvisionRequest) error {
    // ... automated checks ...

    // Wait for human approval (no timeout = wait indefinitely)
    approvalCh := workflow.GetSignalChannel(ctx, "approval")
    var decision ApprovalDecision
    approvalCh.Receive(ctx, &decision)

    if !decision.Approved {
        return fmt.Errorf("provisioning rejected: %s", decision.Reason)
    }

    return workflow.ExecuteActivity(ctx, ProvisionResources, req).Get(ctx, nil)
}
```

The workflow is suspended at the `Receive` call — it consumes no resources while waiting. When an operator approves via the Cadence CLI, SDK, or Web UI, the workflow resumes immediately.

## Real-world examples

- **Uber ticket routing**: [Customer Obsession Ticket Routing Workflow and Orchestration Engine](https://www.uber.com/blog/customer-obsession-ticket-routing-workflow-and-orchestration-engine/)
- **DoorDash financial integrations**: [Enabling Faster Financial Partnership Integrations Using Cadence](https://doordash.engineering/2022/05/18/enabling-faster-financial-partnership-integrations-using-cadence/)
- **Kubernetes cluster provisioning**: [Using Cadence to spin up Kubernetes clusters (Banzai Cloud)](https://github.com/edmondop/cadence-helm-chart)

## Key properties for microservices orchestration

| Requirement | How Cadence handles it |
|---|---|
| **Guaranteed completion** | Workflow state is persisted — survives crashes and restarts |
| **Per-step retry policies** | Each activity has independent retry count, backoff, and timeout |
| **Compensation / rollback** | Explicit compensation activities in error branches (Saga) |
| **Parallel steps** | `workflow.Go` and `workflow.ExecuteActivity` fan out concurrently |
| **Human-in-the-loop** | Signal channels pause execution waiting for external input |
| **Operational visibility** | Full step-by-step status visible in the Cadence Web UI |
| **No polling infrastructure** | No cron jobs, queue workers, or database state machines needed |

## Getting started

- [Cadence Go SDK quickstart](/docs/get-started/)
- [Activity retry policies](/docs/go-client/retries)
- [Signals: human-in-the-loop patterns](/docs/go-client/signals)
- [Child workflows](/docs/go-client/child-workflows)
