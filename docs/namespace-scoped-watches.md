# Plan: namespace-scoped watches and permissions

Status: proposed implementation plan; no runtime changes are included.

This is a separate change from the switches that disable NetworkPolicy and SecretProviderClass management
(`--manage-network-policies`, `--manage-secret-provider-classes`).

## Outcome

Use an explicit namespace list to watch and manage namespaced workload resources.
Grant their read and write permissions through a RoleBinding in each configured
namespace, referencing a shared ClusterRole. Remove cluster-wide read grants for
those resources in restricted mode.

Kubernetes supports a watch in a single namespace. It does not automatically
filter an all-namespaces watch according to the caller's RoleBindings. Therefore
the operator must open one watch per namespace and resource type. The existing
`Api::all` calls cannot remain for namespaced workload resources in restricted mode.

Keep necessary cluster-scoped permissions and the operator namespace's supporting
permissions separate. This change does not make RestateCluster namespaced.

## Configuration and compatibility

Propose an optional `--watch-namespaces` argument and `WATCH_NAMESPACES` environment
variable, encoded as a JSON array to distinguish an explicit empty list from an
unset setting without ambiguous comma/empty-string parsing:

```bash
restate-operator --watch-namespaces='["team-a","team-b"]'
restate-operator --watch-namespaces='[]'
```

| Configuration | Meaning |
| --- | --- |
| Argument/environment unset | Existing all-namespaces behavior, for upstream compatibility |
| Nonempty JSON array | Watch/manage exactly those workload namespaces |
| `[]` | Watch/manage no workload namespaces; valid idle installation |
| Malformed JSON, non-string items, or invalid namespace names | Clear startup configuration error |

Parse once into an explicit `All` or `Restricted(set)` scope, deduplicate names,
and log the effective scope. Never interpret an empty list as all namespaces.
This is startup configuration; changing it requires an operator rollout.

For the Helm chart, propose `namespaceList: null` as the compatible default and
render the environment variable when the value is non-null, including `[]`.
Do not use a truthiness test that omits an empty array and accidentally enables
all-namespaces mode. Update any strict values schema to declare the property.
No validation template or admission webhook is needed.

## Scope by controller

### RestateCluster

In [controller.rs](../src/controllers/restatecluster/controller.rs), replace
all-namespaces watches for these resources with namespace-specific watches:

- StatefulSets, PersistentVolumeClaims, Services, ServiceAccounts, ConfigMaps,
  PodDisruptionBudgets, and NetworkPolicies.
- SecretProviderClasses and supported cloud integration resources when their
  existing availability/enablement conditions are satisfied.
- AWS/GCP integration Jobs, including their special label selectors.

Retain the current selectors, event predicates, owner mapping, and cache contents.
When an optional resource is globally disabled by the `--manage-*` switches, create no
watch for it in any namespace and grant it no permissions.

RestateCluster itself is cluster-scoped. Its `metadata.name` determines the target
namespace. Keep its cluster-scoped watch, but only enqueue/reconcile objects whose
name belongs to the configured namespace list. Add a defensive scope guard at the
start of reconciliation, **before** suspension handling, finalizer helpers,
status writes, events, provisioning, and cleanup. Out-of-scope objects must be
left untouched, including objects with an existing operator finalizer.

Namespace objects are also cluster-scoped. Keep their existing cluster-scoped
watch initially, filtering event-to-owner mapping by the allowed target names.
Namespace get/apply operations must pass the same scope guard. Restrict their
named get/patch RBAC where practical; do not move Namespace permissions into
RoleBindings. Namespace creation permission, if retained, cannot be restricted
with RBAC `resourceNames`. Setups with pre-created namespaces can omit creation.

### RestateDeployment

[This controller](../src/controllers/restatedeployment/controller.rs) also uses
all-namespaces watches. Scoping only RestateCluster would leave its global access
requirements in place. Convert these namespaced sources as part of this change:

- RestateDeployment primary stream and cache.
- ReplicaSets, Services, and HorizontalPodAutoscalers.
- Knative Configurations, Routes, and Revisions when Knative is enabled/detected.

Use each RestateDeployment's own namespace for admission to the managed set.
Apply a defensive guard before finalizer, status, registration, and cleanup work.
Keep cloud environment and credential cache behavior described below.

Audit direct Kubernetes API calls as well as watches. Any namespaced lookup or
mutation derived from a reference must follow the same scope contract or an
explicit operator-namespace exception. Do not silently fall back to `Api::all`.
Registration URLs may intentionally reference an external service or Restate
cluster; the namespace list limits Kubernetes workload management, not outbound
HTTP access. Document this distinction rather than treating the list as network
isolation.

### RestateCloudEnvironment and operator namespace

RestateCloudEnvironment is cluster-scoped and retains its existing cluster-level
handling. Its [controller](../src/controllers/restatecloudenvironment/controller.rs)
already watches tunnel Deployments in the operator namespace. RestateDeployment
already watches authentication Secrets there. Preserve these narrow scopes and
their dedicated RoleBindings.

The operator namespace is not automatically a managed workload namespace. It
receives only the supporting permissions needed there unless explicitly listed.
Cluster-scoped events may require a separate event namespace such as `default`;
audit recorder destinations and retain only the necessary event grants.

An empty list disables workload management, not the separately configured cloud
environment controller. Document that remaining activity explicitly.

## Watch and cache implementation

1. Add the scope type and parsing in [main.rs](../src/main.rs) and propagate it
   through [State and controller contexts](../src/controllers/mod.rs).
2. Introduce shared scope-aware construction for namespaced watch sources, with
   separate all-namespaces and per-namespace paths. Avoid copying scope decisions
   inconsistently across controllers.
3. Preserve cache correctness when combining streams. **Do not feed independent
   raw watcher streams into a single reflector without handling initialization
   and relist boundaries.** One namespace's relist must not clear another
   namespace's objects or mark the entire cache ready prematurely.
4. Prefer separate reflector stores/writers per namespace, with a scoped cache
   abstraction for lookups and iteration, then merge processed object event
   streams for controller routing. Namespace remains part of every object key.
   Adapt existing concrete `Store` consumers, including `Controller::for_stream`,
   deliberately; use per-namespace deployment controllers if that avoids unsafe
   cache aggregation. Share cluster-wide reference caches where appropriate.
5. Preserve prewarming requirements. The current `prewarmed_reflector` supports
   rollback/ownership decisions in RestateDeployment. A controller must not make
   those decisions until its required namespace caches have initialized.
6. With no workload namespaces, construct no workload watchers and treat their
   synchronization as complete. Keep health/readiness responsive and avoid
   waiting forever on a store with no producer. Required CRD/other controller
   readiness checks retain their existing meaning.
7. Treat a 403 in a configured namespace as a visible configuration error. Never
   widen scope or silently remove that namespace. Keep retries bounded by the
   existing watcher backoff and report which namespace/resource failed.

Watch count grows approximately with namespace count times resource types.
Use the explicit list rather than permission probing or scanning RoleBindings.
Test multiple namespaces and startup load before recommending large lists.

## RBAC and chart changes

Split [upstream RBAC](../charts/restate-operator-helm/templates/rbac.yaml) into:

| Permission set | Restricted-mode binding |
| --- | --- |
| Namespaced workload resources, including list/watch | Shared ClusterRole with a RoleBinding in each `namespaceList` entry |
| RestateCluster, RestateCloudEnvironment, Namespace, and required CRD/discovery access | Dedicated cluster-level grants |
| Operator-namespace Secrets, tunnel Deployments, and supporting events | Dedicated RoleBinding in the operator namespace |
| Events in any other required recorder namespace | Dedicated event-only RoleBinding |

Use the operator ServiceAccount namespace in every binding subject, even when the
binding is created in a workload namespace. Deduplicate namespaces and generated
binding names. In compatible all-namespaces mode retain a ClusterRoleBinding for
the workload rule set; omit it entirely in restricted mode.

Keep cluster-scoped list/watch only where still necessary. Restrict RestateCluster
mutation rules by allowed names in restricted mode, and omit those rules when
the list is empty. An absent/empty `resourceNames` restriction must never become
an accidental unrestricted mutation grant. Global reads of cluster-scoped CRs
remain a documented limitation of this first version.

The chart and runtime must derive scope from the same list.
Render the environment setting in the deployment template so list changes roll
the operator. Combine with the optional-resource switches to omit their rules.

## Rollout and namespace lifecycle

- Fresh installation: deploy with `namespaceList: []`, then add the first existing
  namespace and its binding through a chart upgrade.
- Existing installation: add namespace bindings and roll out the scoped binary
  while temporarily retaining previous global grants. After every old operator
  pod has stopped and scoped watches work, remove the global workload grants.
- Adding a namespace: establish permissions before the new operator starts
  watching it. Test Helm/GitOps ordering; do not depend on resource application
  order for correctness.
- Removing a namespace: first finish deletion/draining or arrange transfer of
  management while it is still in scope. Roll out the reduced scope, then revoke
  its bindings. Existing workloads remain; scope removal does not delete them or
  strip finalizers. Unhandled finalizers can otherwise block later CR deletion.
- Returning to all-namespaces mode: restore global permissions before starting
  an operator configured for that mode.

The two-phase transitions may require temporary platform-managed bindings because
a single chart upgrade changes RBAC and the deployment together.

## Verification and acceptance

1. Parser/chart tests distinguish unset, null, empty, one namespace, several
   namespaces, duplicates, and invalid input. An explicit empty list always stays
   restricted.
2. Recording client tests reject all-namespaces endpoints for namespaced workload
   resources in restricted mode. Check startup, retries, reconciliation, finalizer
   paths, and all optional integrations, not only initial watcher construction.
3. Cache tests use identical object names in two namespaces. Relist, deletion, or
   watch restart in one namespace must not evict or confuse the other's objects.
   Verify prewarming, owner routing, and deployment rollback/cleanup behavior.
4. Out-of-scope RestateClusters produce no mutation, event, provisioning, or
   finalizer activity. Out-of-scope RestateDeployments are neither watched nor
   processed. Include direct reconciliation tests to exercise the guards.
5. Add a kind test with two allowed namespaces and one
   excluded namespace. Use a new operator image and only namespace-bound workload
   permissions. Verify all-namespaces workload authorization is denied, allowed
   namespace operations succeed, excluded namespace operations are denied, and
   clusters/deployments become Ready without forbidden watcher errors.
6. Exercise empty installation, adding the first namespace, removing a namespace,
   operator restart, and a missing RoleBinding. Check readiness does not hang for
   an intentional empty scope and that permission failures remain visible.
7. Combine both optional-resource disable flags with restricted scope. With the
   CSI CRD present, assert no NetworkPolicy or SecretProviderClass requests and
   no corresponding effective permissions. Exercise operator-namespace cloud
   credential/tunnel access separately from workload permissions.
8. Run the existing all-namespaces regression suite and relevant Rust checks.
   Include Knative/cloud watch-construction coverage even when the local kind
   environment does not contain those integrations.

Acceptance: in restricted mode, every namespaced workload API request targets an
explicitly allowed namespace, no workload watcher needs global resource access,
and both allowed namespaces reconcile correctly with namespace RoleBindings only.

## Delivery

Implement scope parsing and cache/watch support first, then controller guards,
Helm/RBAC wiring, and integration tests. Ship them together with documentation and
a feature release note. No CRD scope or generated schema change is needed.
Keep the optional-resource flags as a separate feature with combined tests.

Further reduction of cluster-scoped CR reads, dynamic scope reload, and automatic
permission discovery are outside this plan.
