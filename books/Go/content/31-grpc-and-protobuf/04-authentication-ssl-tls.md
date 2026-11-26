---
title: '04. Authentication, SSL/TLS'
---

# Authentication, SSL/TLS

Plaintext gRPC (`insecure.NewCredentials()`) is acceptable only on loopback during local development. Production gRPC services encrypt transport traffic with **server-side TLS** and, for zero-trust service-to-service communication, authenticate both peers with **mutual TLS (mTLS)**.

## Module Syllabus (5 Lessons)

1. What is SSL/TLS?
2. Implementing Server-side TLS in gRPC
3. Implementing mTLS in gRPC
4. Exercise: Add mTLS to Your File Upload Application
5. Exercise Solution

## Server-Side TLS vs. mTLS

```text
Server-Side TLS:
  Client verifies Server Cert against trusted CA  --->  Encrypted channel (Client identity unchecked)

Mutual TLS (mTLS):
  Client verifies Server Cert against CA
  AND Server verifies Client Cert against CA      --->  Encrypted + mutually authenticated channel
```

## Configuring mTLS in Go gRPC

```go
package tlscfg

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"os"

	"google.golang.org/grpc/credentials"
)

func NewServerMTLSCreds(certFile, keyFile, clientCAFile string) (credentials.TransportCredentials, error) {
	serverCert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return nil, fmt.Errorf("load server key pair: %w", err)
	}
	caPEM, err := os.ReadFile(clientCAFile)
	if err != nil {
		return nil, fmt.Errorf("read client CA: %w", err)
	}
	clientCAs := x509.NewCertPool()
	if !clientCAs.AppendCertsFromPEM(caPEM) {
		return nil, fmt.Errorf("failed to append client CA certificate")
	}

	cfg := &tls.Config{
		MinVersion:   tls.VersionTLS13,
		Certificates: []tls.Certificate{serverCert},
		ClientAuth:   tls.RequireAndVerifyClientCert,
		ClientCAs:    clientCAs,
	}
	return credentials.NewTLS(cfg), nil
}

func NewClientMTLSCreds(certFile, keyFile, serverCAFile, serverName string) (credentials.TransportCredentials, error) {
	clientCert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return nil, fmt.Errorf("load client key pair: %w", err)
	}
	caPEM, err := os.ReadFile(serverCAFile)
	if err != nil {
		return nil, fmt.Errorf("read server CA: %w", err)
	}
	rootCAs := x509.NewCertPool()
	if !rootCAs.AppendCertsFromPEM(caPEM) {
		return nil, fmt.Errorf("failed to append server CA certificate")
	}

	cfg := &tls.Config{
		MinVersion:   tls.VersionTLS13,
		Certificates: []tls.Certificate{clientCert},
		RootCAs:      rootCAs,
		ServerName:   serverName,
	}
	return credentials.NewTLS(cfg), nil
}
```

Pass the returned credentials via `grpc.Creds(serverCreds)` on the server and `grpc.WithTransportCredentials(clientCreds)` on `grpc.NewClient(...)`.

## Related Chapters

- Previous: [03. gRPC Streaming](03-grpc-streaming.md)
- Next: [05. Interceptors, Metadata & Authorization](05-interceptors-metadata-authz.md)
- Book foundations: [TLS & mTLS Deep Dive](../22-security-hardening/171-tls-mtls-deep-dive.md)
