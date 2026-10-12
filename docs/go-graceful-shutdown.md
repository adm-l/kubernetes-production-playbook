# Go HTTP graceful shutdown on Kubernetes

Kubernetes normally sends SIGTERM before stopping a container, then force-kills it after the Pod termination grace period. The application must cooperate.

## Example

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok"))
	})
	mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
		// In a real app, reflect whether the process is accepting work
		// and whether required local initialization has completed.
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ready"))
	})

	srv := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
	}

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	errCh := make(chan error, 1)
	go func() {
		log.Printf("listening on %s", srv.Addr)
		errCh <- srv.ListenAndServe()
	}()

	select {
	case err := <-errCh:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("HTTP server failed: %v", err)
		}
	case <-ctx.Done():
		// Give active requests time to finish. This timeout must fit inside
		// terminationGracePeriodSeconds, leaving a margin for process exit.
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
		defer cancel()
		if err := srv.Shutdown(shutdownCtx); err != nil {
			log.Printf("graceful shutdown timed out: %v", err)
			_ = srv.Close()
		}
	}
}
```

## Production notes
- The example is a teaching baseline, not a complete application. Add dependency cleanup and error handling for your server.
- Keep shutdown timeout shorter than Pod termination grace period; reserve time for preStop hooks and process exit if used.
- Readiness should turn false when the process should no longer receive new traffic. Coordinate with endpoint removal and load-balancer draining.
- Do not sleep blindly as a substitute for a well-designed readiness and shutdown flow.
- Test SIGTERM under load and verify in-flight requests, queue consumers and DB connections are handled correctly.
- For background consumers, stop claiming new messages, finish or safely release in-flight work, then commit offsets/acknowledge according to the messaging system's delivery semantics.
