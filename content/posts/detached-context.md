+++
date = '2025-05-05T14:54:42+02:00'
draft = false
title = 'Detached Context'
+++


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

1. noCancelCTX := nocancel.FromContext(ctx)
It takes an existing context.Context (ctx) and returns a new context that is not canceled when the original one is. This is useful when you want to detach a background task from the lifecycle of the incoming request.
✅ Why use it?
If ctx is tied to an HTTP request or RPC call, it will be canceled when the client disconnects or the request times out.By using nocancel.FromContext(ctx), you're saying:
> "I want this background task to keep running even if the original request is canceled."

2. newContext, cancelFunc := context.WithTimeout(noCancelCTX, time.Duration(10)*time.Minute)
This line creates a new context with a timeout of 10 minutes.
* newContext is a child of noCancelCTX, and it will be automatically canceled after 10 minutes.
* cancelFunc is a function you must call (usually with defer cancelFunc()) to release resources early if you're done before the timeout.
✅ Why use it?
Even though you've detached from the original context, you still want to limit how long the background task runs, to avoid runaway processes.

> "Create a context that is not canceled when the original request ends, but will automatically cancel after 10 minutes to avoid running forever."


🧪 Example Use Case
Imagine a user triggers a background re-enrollment process via an API call. You want the process to continue even if the user closes the browser — but you also don’t want it to run forever.
This pattern ensures:
* ✅ The task survives client disconnects.
* ✅ The task still has a hard timeout (10 minutes).
* ✅ You can cancel it manually if needed.