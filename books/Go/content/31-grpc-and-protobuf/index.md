---
title: Intro
---

# Intro

This part is a complete, production-oriented Go curriculum for **Protocol Buffers, gRPC, and ConnectRPC**—progressing from `.proto` contracts and `protoc` code generation to streaming RPCs, TLS/mTLS, interceptors, client-side load balancing, testing, Kubernetes deployment, and modern Protobuf management with **Buf**.

## Curriculum Architecture & Progression

```text
01. Protobuf, gRPC & Protoc -> 02. Simple gRPC Service (TODO App) -> 03. Streaming (File Upload)
        |
        v
04. Authentication & SSL/TLS (mTLS) -> 05. Interceptors, Metadata & Auth (API Keys, JWT)
        |
        v
06. Client Service Config & Load Balancing -> 07. Testing gRPC Services (Postman, gRPCurl, Go Tests)
        |
        v
08. Containerising & Deploying to Kubernetes -> 09. Buf, ConnectRPC & Protovalidate
```

## Module Syllabus & Chapter Map

| # | Module | Lessons | Core Focus | Chapter |
|---|--------|---------|------------|---------|
| 01 | **Introduction to Protobuf, gRPC & Protoc** | 4 | Protobuf benefits, gRPC overview, 4 RPC types, `protoc` codegen | [01. Introduction to Protobuf, gRPC & Protoc](01-protobuf-grpc-protoc.md) |
| 02 | **Building a Simple gRPC Service** | 7 | `.proto` contracts, server/client, gRPC status errors, TODO app, clean server lifecycle | [02. Building a Simple gRPC Service](02-simple-grpc-service.md) |
| 03 | **gRPC Streaming** | 5 | Server, client, and bi-directional streaming; file upload application | [03. gRPC Streaming](03-grpc-streaming.md) |
| 04 | **Authentication, SSL/TLS** | 5 | SSL/TLS fundamentals, server-side TLS, mutual TLS (mTLS) | [04. Authentication, SSL/TLS](04-authentication-ssl-tls.md) |
| 05 | **Interceptors, Metadata & Authorization** | 7 | Unary/stream interceptors, deadlines, metadata, API keys, `CallCredentials`, JWT | [05. Interceptors, Metadata & Authorization](05-interceptors-metadata-authz.md) |
| 06 | **Client Service Config & Load Balancing** | 5 | Service config, automatic retries, round-robin & custom LB policies | [06. Client Service Config & Load Balancing](06-client-service-config-lb.md) |
| 07 | **Testing gRPC Services** | 8 | Postman, `grpcurl`, unit & E2E tests for unary and streaming RPCs | [07. Testing gRPC Services](07-testing-grpc-services.md) |
| 08 | **Containerising & Deploying to Kubernetes** | 6 | Docker, Kubernetes, Let's Encrypt TLS, LoadBalancer, Ingress, Cloudflare Tunnel | [08. Containerising & Deploying to Kubernetes](08-containerising-deploying-k8s.md) |
| 09 | **Using Buf to Manage Protobuf** | 7 | Buf at scale, BSR, ConnectRPC (Go & Web), Protovalidate | [09. Using Buf to Manage Protobuf](09-buf-and-connectrpc.md) |

## Related Parts in This Book

- [gRPC Microservices Overview](../16-web/62-grpc-microservices.md)
- [TLS & mTLS Deep Dive](../22-security-hardening/171-tls-mtls-deep-dive.md)
- [Kubernetes Deployment Basics](../20-infrastructure/75-kubernetes-deploy-basics.md)
