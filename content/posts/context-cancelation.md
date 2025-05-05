+++
date = '2025-05-05T15:09:35+02:00'
draft = false
title = 'Context Cancelation'
+++

In previous post we have a code:

```go
func (s *service) DoSomethingWithContext(
	ctx context.Context,
	idA string,
	idB string,
) (*model.SomeStatus, error) {
	req := someapi.TriggerRequest{
		FieldA: idA,
		FieldB: idB,
	}
	noCancelCTX := nocancel.FromContext(ctx)

	go func() {
		// Defer cancelFunc to ensure the context lives through the RPC
		newCtx, cancel := context.WithTimeout(noCancelCTX, 10*time.Minute)
		defer cancel()

		resp, err := s.client.TriggerAction(
			newCtx,
			connect.NewRequest(&req),
		)
		if err != nil {
			telemetry.L(ctx).Error("Async action failed", zap.Error(err))
		} else {
			telemetry.L(ctx).Info("Async action succeeded", zap.Any("status", model.SomeStatus(resp.Msg.Status)))
		}
	}()

	return ptr.To(model.SomeStatusStarted), nil
}

```

Why do we need to call defer cancel() ?
In Go, when you create a context with context.WithTimeout, context.WithCancel or context.WithDeadline, the runtime sets up internal resources like timers and cancellation signals to track its lifecylce.

🧠 What Resources Does Go Allocate for a Context?
When you create a cancellable context (e.g., with context.WithTimeout), Go internally sets up:
1. A Timer
* Go starts a timer that will fire when the timeout or deadline is reached.
* This timer is managed by the Go runtime and consumes memory and scheduling resources.
2. A Goroutine (sometimes)
* In some cases, Go may spawn a goroutine to monitor the timer and propagate cancellation to child contexts.
* This goroutine waits for the timer to expire or for cancelFunc() to be called.
3. Cancellation Channels
* Each context has a Done() channel that is closed when the context is canceled.
* This channel is used to notify all downstream consumers that they should stop work.

✅ Why cancelFunc Matters
Calling cancelFunc():
* Stops the timer early (if it hasn’t fired yet).
* Signals cancellation to all child contexts and goroutines using this context.
* Allows Go to clean up the timer, goroutine, and internal data structures immediately, instead of waiting for the timeout.

🧪 Example
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

Copied
If your operation finishes in 100ms, calling cancel():
* Stops the 10-second timer.
* Frees the memory and runtime resources associated with it.
* Prevents a goroutine from sitting idle for 9.9 more seconds.

✅ Summary
> Go allocates timers, channels, and sometimes goroutines to manage context cancellation.> Calling cancelFunc() ensures those resources are cleaned up immediately, avoiding leaks and unnecessary work.