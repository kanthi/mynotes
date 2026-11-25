---
title: '04. Durable Workflows with Temporal'
---

# Durable Workflows with Temporal

Hand-rolled [Sagas and Process Managers](../29-event-driven-dev/20-sagas-and-process-managers.md) work well for 2–4 step flows, but long-running workflows with timers, human approvals, retries, and compensations across days or weeks are far simpler with a **durable execution engine** like **Temporal**.

## How Durable Execution Works

- **Workflows** are deterministic Go functions whose execution history is persisted as an event log. If a worker crashes mid-workflow, another worker replays the history and resumes at the exact line where it left off.
- **Activities** perform all non-deterministic side effects (HTTP calls, DB writes, emails) with automatic configurable retries and heartbeats.

## Workflow with Compensating Rollback (Saga) in Go

```go
package workflow

import (
	"time"

	"go.temporal.io/sdk/temporal"
	"go.temporal.io/sdk/workflow"
)

func BookVIPBundleWorkflow(ctx workflow.Context, bundleID string) error {
	ao := workflow.ActivityOptions{
		StartToCloseTimeout: 30 * time.Second,
		RetryPolicy: &temporal.RetryPolicy{
			MaximumAttempts: 5,
		},
	}
	ctx = workflow.WithActivityOptions(ctx, ao)

	var bookingID string
	if err := workflow.ExecuteActivity(ctx, "ReserveTickets", bundleID).Get(ctx, &bookingID); err != nil {
		return err
	}

	if err := workflow.ExecuteActivity(ctx, "BookTaxi", bundleID).Get(ctx, nil); err != nil {
		// Compensate: refund tickets if taxi booking fails.
		_ = workflow.ExecuteActivity(ctx, "RefundTickets", bookingID).Get(ctx, nil)
		return err
	}
	return nil
}
```

## Determinism Rules

Inside a Temporal Workflow function, never call `time.Now()`, `math/rand`, or iterate over Go `map`s directly—use `workflow.Now(ctx)`, `workflow.SideEffect`, and sorted slices so replay always produces identical decisions.
