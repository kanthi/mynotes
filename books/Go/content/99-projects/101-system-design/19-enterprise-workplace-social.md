---
title: 'Design Lab: Enterprise Workplace Social Network ("Facebook for Office")'
---

# Design Lab: Enterprise Workplace Social Network ("Facebook for Office")

An enterprise workplace social network (in the style of Workplace, Viva Engage, or an internal company social hub) looks like a consumer social feed on the surface, but its **security, identity, and visibility invariants are radically stricter**:

- **Zero cross-tenant leakage**: Company A and Company B share infrastructure, yet one missing `WHERE tenant_id = $1` clause in a feed query or search filter is a catastrophic security incident.
- **Instant offboarding**: When HR terminates an employee in Okta or Entra ID via [SCIM 2.0 (RFC 7644)](https://www.rfc-editor.org/info/rfc7644/), active sessions, WebSockets, and API tokens must revoke in under a second—not wait for a 1-hour JWT expiry.
- **Strict group ACLs**: Posts in a `SECRET` M&A or HR group must never appear in "All-Company" feeds, trending lists, or full-text search results for non-members—even if an employee was removed from the group 2 seconds ago.
- **Executive broadcasts without write storms**: A CEO posting to an All-Company group of 80,000 employees must not trigger 80,000 synchronous database writes.

![Enterprise workplace social network architecture with SSO/OIDC and SCIM directory sync, tenant-scoped API gateway, Go services for Org Graph, Groups, Hybrid Feed, Live Presence, Search, and Compliance Audit backed by PostgreSQL RLS, Redis, Kafka, and Elasticsearch](../../../images/diagram-go-enterprise-workplace-social.svg)

## Architecture and module layout

Build the platform as a clean multi-tenant Go module with PostgreSQL [Row-Level Security (RLS)](https://www.postgresql.org/docs/current/ddl-rowsecurity.html), Redis Cluster, Kafka, and Elasticsearch/OpenSearch:

```text
workplace-social/
├── cmd/
│   ├── gateway/main.go         # OIDC JWT verification + Redis revocation check
│   ├── api/main.go             # Org graph, groups, posts, hybrid feed, SCIM API
│   └── workers/main.go         # Outbox relay, feed fan-out, ES ACL indexer, audit sink
├── internal/
│   ├── tenant/                 # WithTenantTx (SET LOCAL app.tenant_id) + context keys
│   ├── identity/               # OIDC verifier + SCIM 2.0 user/group provisioning
│   ├── groups/                 # PUBLIC | CLOSED | SECRET group membership & ACLs
│   ├── feed/                   # Hybrid push/pull feed with live ACL re-check
│   ├── search/                 # Per-tenant ACL-tagged Elasticsearch queries
│   └── compliance/             # Immutable WORM audit log + legal hold retention
└── migrations/001_workplace.sql
```

## Multi-tenant schema with PostgreSQL Row-Level Security

Enforce tenant isolation **inside PostgreSQL** using `FORCE ROW LEVEL SECURITY` on a non-superuser application role. Even if a developer forgets a `tenant_id` predicate in a new query, the database refuses to return or modify another company's rows.

```sql
CREATE TABLE tenants (
    id          UUID PRIMARY KEY,
    slug        TEXT UNIQUE NOT NULL,
    oidc_issuer TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employees (
    id          UUID PRIMARY KEY,
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    external_id TEXT NOT NULL, -- SCIM externalId from Okta / Entra ID
    email       TEXT NOT NULL,
    department  TEXT NOT NULL,
    active      BOOLEAN NOT NULL DEFAULT true,
    revoked_at  TIMESTAMPTZ,
    UNIQUE (tenant_id, external_id)
);

CREATE TABLE groups (
    id          UUID PRIMARY KEY,
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    name        TEXT NOT NULL,
    privacy     TEXT NOT NULL, -- OPEN | CLOSED | SECRET
    is_all_co   BOOLEAN NOT NULL DEFAULT false,
    member_cnt  INT NOT NULL DEFAULT 0
);

CREATE TABLE group_members (
    tenant_id   UUID NOT NULL,
    group_id    UUID NOT NULL REFERENCES groups(id),
    employee_id UUID NOT NULL REFERENCES employees(id),
    role        TEXT NOT NULL DEFAULT 'MEMBER', -- MEMBER | MODERATOR | ADMIN
    PRIMARY KEY (tenant_id, group_id, employee_id)
);

CREATE TABLE posts (
    id          BIGINT PRIMARY KEY, -- Time-ordered Snowflake ID
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    group_id    UUID NOT NULL REFERENCES groups(id),
    author_id   UUID NOT NULL REFERENCES employees(id),
    body        TEXT NOT NULL,
    legal_hold  BOOLEAN NOT NULL DEFAULT false,
    deleted_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Enforce tenant isolation at the storage engine level.
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
ALTER TABLE employees FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_iso_employees ON employees
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);

ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE posts FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_iso_posts ON posts
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

## Go implementation: SCIM instant offboarding and ACL-safe hybrid feed

### 1. Instant offboarding via SCIM 2.0 (`DELETE /scim/v2/Users/{id}`)

Stateless JWTs remain valid until `exp`. To achieve sub-second enterprise offboarding, SCIM deactivation updates `employees.active = false`, sets a revocation timestamp in Redis (`revoked:{tenant_id}:{employee_id}`), and publishes a `UserRevoked` event on Redis Pub/Sub so every WebSocket/SSE gateway immediately drops that employee's open streams.

```go
package identity

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

type RevocationStore interface {
	SetRevokedAfter(ctx context.Context, tenantID, employeeID string, ts time.Time) error
	PublishDisconnect(ctx context.Context, tenantID, employeeID string) error
}

// DeactivateEmployee handles SCIM 2.0 offboarding atomically: marks the employee
// inactive in PostgreSQL, records an immutable audit event, revokes active JWTs
// in Redis, and kicks open WebSocket/SSE connections across all gateway pods.
func DeactivateEmployee(ctx context.Context, db *sql.DB, rev RevocationStore, tenantID, externalID, actor string) (string, error) {
	now := time.Now().UTC()

	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return "", err
	}
	defer tx.Rollback()

	// Bind PostgreSQL RLS to this tenant for the duration of the transaction.
	if _, err := tx.ExecContext(ctx, `SELECT set_config('app.tenant_id', $1, true)`, tenantID); err != nil {
		return "", err
	}

	var empID string
	err = tx.QueryRowContext(ctx, `
		UPDATE employees
		SET active = false, revoked_at = $1
		WHERE external_id = $2
		RETURNING id`, now, externalID,
	).Scan(&empID)
	if err != nil {
		return "", fmt.Errorf("scim deactivate %s: %w", externalID, err)
	}

	if _, err := tx.ExecContext(ctx, `
		INSERT INTO compliance_audit_log (tenant_id, actor, action, target_id, occurred_at)
		VALUES ($1, $2, 'SCIM_USER_DEACTIVATED', $3, $4)`,
		tenantID, actor, empID, now,
	); err != nil {
		return "", err
	}

	if err := tx.Commit(); err != nil {
		return "", err
	}

	// Propagate revocation to Redis & active real-time gateways immediately.
	if err := rev.SetRevokedAfter(ctx, tenantID, empID, now); err != nil {
		return empID, err
	}
	_ = rev.PublishDisconnect(ctx, tenantID, empID)
	return empID, nil
}
```

### 2. ACL-enforced hybrid workplace feed

Unlike consumer social networks, an enterprise feed mixes **team groups** (50–500 members -> fan-out-on-write to Redis timelines) with **All-Company / Division groups** (10,000–100,000 members -> fan-out-on-read merged lazily). Every hydrated post is checked against the caller's live `group_members` set before returning:

```go
package feed

import (
	"context"
	"sort"
	"time"
)

const LargeGroupThreshold = 5_000

type Post struct {
	ID        int64
	TenantID  string
	GroupID   string
	Privacy   string // OPEN | CLOSED | SECRET
	Body      string
	Deleted   bool
	CreatedAt time.Time
}

type ACLStore interface {
	MemberGroupIDs(ctx context.Context, tenantID, employeeID string) (map[string]bool, error)
	LargeGroupIDs(ctx context.Context, tenantID string) ([]string, error)
	PrecomputedPostIDs(ctx context.Context, tenantID, employeeID string, limit int) ([]int64, error)
	RecentLargeGroupPosts(ctx context.Context, tenantID string, groupIDs []string, limit int) ([]Post, error)
	BatchLoadPosts(ctx context.Context, tenantID string, ids []int64) ([]Post, error)
}

// BuildHomeFeed merges precomputed team-group posts with All-Company pull posts
// and enforces strict live group ACLs so removed members never see SECRET posts.
func BuildHomeFeed(ctx context.Context, s ACLStore, tenantID, employeeID string, limit int) ([]Post, error) {
	if limit <= 0 || limit > 50 {
		limit = 20
	}
	memberships, err := s.MemberGroupIDs(ctx, tenantID, employeeID)
	if err != nil {
		return nil, err
	}

	pushIDs, err := s.PrecomputedPostIDs(ctx, tenantID, employeeID, limit*2)
	if err != nil {
		return nil, err
	}
	pushPosts, err := s.BatchLoadPosts(ctx, tenantID, pushIDs)
	if err != nil {
		return nil, err
	}

	largeGroups, err := s.LargeGroupIDs(ctx, tenantID)
	if err != nil {
		return nil, err
	}
	var allowedLarge []string
	for _, gid := range largeGroups {
		if memberships[gid] {
			allowedLarge = append(allowedLarge, gid)
		}
	}
	pullPosts, err := s.RecentLargeGroupPosts(ctx, tenantID, allowedLarge, limit*2)
	if err != nil {
		return nil, err
	}

	seen := make(map[int64]bool, len(pushPosts)+len(pullPosts))
	candidates := make([]Post, 0, len(pushPosts)+len(pullPosts))
	for _, p := range append(pushPosts, pullPosts...) {
		if seen[p.ID] || p.Deleted || p.TenantID != tenantID {
			continue
		}
		// Mandatory live ACL check: CLOSED and SECRET groups require active membership.
		if p.Privacy != "OPEN" && !memberships[p.GroupID] {
			continue
		}
		seen[p.ID] = true
		candidates = append(candidates, p)
	}

	sort.Slice(candidates, func(i, j int) bool {
		if !candidates[i].CreatedAt.Equal(candidates[j].CreatedAt) {
			return candidates[i].CreatedAt.After(candidates[j].CreatedAt)
		}
		return candidates[i].ID > candidates[j].ID
	})
	if len(candidates) > limit {
		candidates = candidates[:limit]
	}
	return candidates, nil
}
```

## Enterprise search ACLs, compliance, and capacity math

- **ACL-filtered search**: Tag every Elasticsearch document with `tenant_id`, `group_id`, and `privacy`. At query time, inject a mandatory filter `(tenant_id == T) AND (privacy == "OPEN" OR group_id IN caller_memberships)`—never trust client-supplied search filters.
- **Legal hold & retention**: When a post or group is under `legal_hold = true`, user deletion sets `deleted_at` (hiding it from the UI and search) while retaining the immutable row and media blob in WORM S3 storage for eDiscovery export.
- **Capacity (500 enterprise tenants, 2M total employees, 400k DAU)**: ~150k posts/day (`~1.7/s` avg, `~20/s` town-hall peak), ~10M feed reads/day (`~115 req/s` avg, `~1,200 req/s` peak during Monday morning / CEO live Q&A). The hybrid All-Company pull path keeps fan-out writes under 10k/s even during 80,000-employee announcements.

## Acceptance drills (`go test -race ./...`)

1. **Cross-tenant RLS isolation**: Seed Tenant A and Tenant B in the same PostgreSQL database; run a raw `SELECT * FROM posts` without a `WHERE` clause inside `app.tenant_id = A` and assert zero Tenant B rows are returned.
2. **Instant SCIM revocation**: Authenticate Employee X, connect an SSE feed stream, invoke `DeactivateEmployee`, and assert the SSE stream closes and the next HTTP request with X's unexpired JWT returns `401 Unauthorized`.
3. **Secret group revocation**: Post to a `SECRET` group, fan out to Member Y's Redis timeline, remove Y from `group_members`, and assert `BuildHomeFeed` returns zero posts from that group.
4. **Town-hall spike**: Publish 1 post to an All-Company group of 50,000 employees and verify publish latency stays `< 50 ms` with zero push fan-out storm.

## Reference materials

- [RFC 7644 — SCIM 2.0 Provisioning Protocol](https://www.rfc-editor.org/info/rfc7644/)
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/info/rfc6749/)
- [PostgreSQL Row Security Policies (`ddl-rowsecurity`)](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [News feed fan-out and cursors lab](03-news-feed.md) and [AuthN/AuthZ patterns](../../22-security-hardening/172-authn-authz-patterns.md)
