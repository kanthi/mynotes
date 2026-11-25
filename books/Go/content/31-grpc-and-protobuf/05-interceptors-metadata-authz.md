---
title: '05. Interceptors, Metadata & Authorization'
---

# Interceptors, Metadata & Authorization

**Interceptors** are gRPC's equivalent of HTTP middleware—running around every unary or streaming RPC for logging, metrics, panic recovery, and authentication. **Metadata** carries key-value headers (like `authorization` or `x-request-id`) over HTTP/2 frames, while **deadlines** propagate timeouts end to end.

## Module Syllabus (7 Lessons)

1. Interceptors — Introduction, Client Interceptors, Server Interceptors
2. Setting Deadlines/Timeouts
3. `CallOptions` & Metadata
4. API Key Authorization via Interceptors
5. Auth via `PerRPCCredentials` / `CallCredentials`
6. Exercise: Add Interceptor to Parse JWT Information
7. Exercise Solution

## Server Unary Interceptor for Authorization & JWT Context

```go
package auth

import (
	"context"
	"strings"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

type claimsKey struct{}

type UserClaims struct {
	Subject string
	Role    string
}

func ClaimsFromContext(ctx context.Context) (UserClaims, bool) {
	c, ok := ctx.Value(claimsKey{}).(UserClaims)
	return c, ok
}

func JWTUnaryInterceptor(verify func(token string) (UserClaims, error)) grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req any,
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (any, error) {
		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return nil, status.Error(codes.Unauthenticated, "missing metadata")
		}
		vals := md.Get("authorization")
		if len(vals) == 0 || !strings.HasPrefix(vals[0], "Bearer ") {
			return nil, status.Error(codes.Unauthenticated, "missing bearer token")
		}

		claims, err := verify(strings.TrimPrefix(vals[0], "Bearer "))
		if err != nil {
			return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
		}

		ctx = context.WithValue(ctx, claimsKey{}, claims)
		return handler(ctx, req)
	}
}
```

## Client `PerRPCCredentials` & Deadlines

Implement `credentials.PerRPCCredentials` so the gRPC client attaches credentials automatically on every RPC (or per-call via `grpc.PerRPCCredentials(...)` `CallOption`):

```go
type APIKeyCreds struct {
	Key string
}

func (a APIKeyCreds) GetRequestMetadata(ctx context.Context, _ ...string) (map[string]string, error) {
	return map[string]string{"x-api-key": a.Key}, nil
}

func (a APIKeyCreds) RequireTransportSecurity() bool {
	return true // refuses to send secrets over plaintext connections
}
```

Always set a deadline on client calls with `context.WithTimeout(ctx, 3*time.Second)`—gRPC encodes the remaining budget in the `grpc-timeout` header so downstream services automatically stop work when the caller times out.

## Related Chapters

- Previous: [04. Authentication, SSL/TLS](04-authentication-ssl-tls.md)
- Next: [06. Client Service Config & Load Balancing](06-client-service-config-lb.md)
- Book foundations: [JWT and Token Validation](../22-security-hardening/174-jwt-and-token-validation.md)
