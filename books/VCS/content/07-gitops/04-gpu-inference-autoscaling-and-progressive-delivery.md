---
title: 'Project: GPU Inference Autoscaling & Argo Rollouts Progressive Delivery'
---

# Project: GPU Inference Autoscaling & Argo Rollouts Progressive Delivery

Combine two high-demand platform engineering patterns:
1. **GPU & Event-Driven Inference Autoscaling:** Deploy an open-weight LLM via **vLLM / KServe** on Kubernetes with **Karpenter** node provisioning and **KEDA** scaling replicas on custom Prometheus request-queue depth metrics, paired with SLO error-budget burn alerts.
2. **Argo Rollouts Canary Progressive Delivery:** Gate progressive traffic shifts (`10% -> 30% -> 60% -> 100%`) with automated Prometheus `AnalysisTemplate` error-rate and latency queries, plus automatic rollback on regression.

## KEDA ScaledObject for vLLM Queue Depth

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-inference-scaler
spec:
  scaleTargetRef:
    name: vllm-llama
  minReplicaCount: 1
  maxReplicaCount: 8
  cooldownPeriod: 180
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring:9090
        metricName: vllm_num_requests_waiting
        query: sum(vllm:num_requests_waiting{model="llama-3"})
        threshold: "4"
```

## Argo Rollouts Canary Analysis Gate

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-gate
spec:
  metrics:
    - name: http-success-rate
      interval: 30s
      successCondition: result[0] >= 0.99
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring:9090
          query: |
            sum(rate(http_requests_total{app="inference",status!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{app="inference"}[2m]))
```

## Acceptance Drills

- Drive synthetic concurrent prompts against vLLM and observe KEDA scaling replicas up as `vllm:num_requests_waiting` crosses threshold, then scaling down cleanly after `cooldownPeriod`.
- Inject a 5% HTTP 500 fault into the canary revision and confirm Argo Rollouts automatically aborts and restores 100% traffic to stable.
