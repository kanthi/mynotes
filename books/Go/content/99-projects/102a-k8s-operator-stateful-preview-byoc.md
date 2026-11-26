---
title: '102a. Project: Go Kubernetes Operator for Ephemeral Previews, Stateful Failover & BYOC'
---

# Project: Go Kubernetes Operator for Ephemeral Previews, Stateful Failover & BYOC

Build a production-grade Kubernetes Operator in Go using **Kubebuilder / `controller-runtime`** that reconciles two real-world platform patterns: **multi-tenant ephemeral preview environments** (with zero-trust NetworkPolicies, Gateway API routes, optional gVisor/Kata `RuntimeClass`, and TTL cleanup) and **stateful BYOC ("Bring Your Own Cloud") clusters** (with leader election, CSI volume snapshots, mTLS rotation, and validating admission webhooks).

## Architecture

```text
Git PR / Customer Helm Install
        |
        v
Custom Resources (PreviewEnvironment / ManagedCluster)
        |
        v  Validating Webhook (rejects unsafe quotas, missing TTL, root securityContext)
[ Go Controller-Runtime Operator (Leader-Elected, Prometheus /metrics) ]
   ├── Reconcile Namespace + ResourceQuota + Cilium/NetworkPolicy (default-deny)
   ├── Reconcile Gateway API HTTPRoute + TLS cert-manager Certificate
   ├── Reconcile StatefulSet (optional gVisor/Kata RuntimeClass) + CSI VolumeSnapshot
   └── Reconcile TTL Finalizer -> clean teardown on expiration or PR close
```

## Core CRD & Reconcile Loop Skeleton

```go
package controller

import (
	"context"
	"time"

	corev1 "k8s.io/api/core/v1"
	netv1 "k8s.io/api/networking/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

const previewFinalizer = "platform.example.com/cleanup"

type PreviewEnvironmentSpec struct {
	PRNumber     int    `json:"prNumber"`
	Image        string `json:"image"`
	RuntimeClass string `json:"runtimeClass,omitempty"` // e.g., "gvisor" or "kata"
	TTLMinutes   int32  `json:"ttlMinutes"`
}

type PreviewReconciler struct {
	client.Client
}

func (r *PreviewReconciler) reconcileNamespaceAndIsolation(ctx context.Context, name string) error {
	ns := &corev1.Namespace{ObjectMeta: metav1.ObjectMeta{Name: name}}
	if _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ns, func() error {
		if ns.Labels == nil {
			ns.Labels = map[string]string{}
		}
		ns.Labels["pod-security.kubernetes.io/enforce"] = "restricted"
		return nil
	}); err != nil {
		return err
	}

	// Default-deny ingress/egress NetworkPolicy for tenant isolation.
	np := &netv1.NetworkPolicy{ObjectMeta: metav1.ObjectMeta{Name: "default-deny", Namespace: name}}
	_, err := controllerutil.CreateOrUpdate(ctx, r.Client, np, func() error {
		np.Spec = netv1.NetworkPolicySpec{
			PodSelector: metav1.LabelSelector{},
			PolicyTypes: []netv1.PolicyType{netv1.PolicyTypeIngress, netv1.PolicyTypeEgress},
		}
		return nil
	})
	return err
}

func computeRequeueAfter(created time.Time, ttlMinutes int32) (expired bool, wait time.Duration) {
	deadline := created.Add(time.Duration(ttlMinutes) * time.Minute)
	remaining := time.Until(deadline)
	if remaining <= 0 {
		return true, 0
	}
	return false, remaining
}

var _ = apierrors.IsNotFound
var _ = ctrl.Result{}
```

## Production Packaging & Verification

- **BYOC Helm Chart:** Package the operator, CRDs, RBAC, and ServiceMonitor in a Helm chart that runs in locked-down customer VPCs with `imagePullSecrets`, private registry overrides, and `cert-manager` mTLS rotation.
- **GPU-Aware Scheduler Extender (Optional Extension):** Add a webhook/extender that inspects simulated GPU memory availability and falls back to secondary node pools when capacity saturates.
- **Acceptance Drills:** Run `envtest` integration tests for creation, TTL expiration, finalizer cleanup, and webhook rejection; verify leader-election failover by killing the active operator pod.
