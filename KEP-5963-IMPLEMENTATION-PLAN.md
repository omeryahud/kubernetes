# KEP-5963: DRA Device Compatibility Groups — Alpha Implementation Plan

## Context

**KEP-5963** ([PR #5964](https://github.com/kubernetes/enhancements/pull/5964), authored by @omeryahud, owning SIG `sig-node`, participating `sig-scheduling`) adds a new `compatibilityGroups []string` field to each `DeviceCounterConsumption` entry inside a `ResourceSlice`. It lets DRA drivers declare which partitioned devices are mutually *compatible* when they consume from the same `CounterSet`.

**Problem.** Today, a DRA driver that partitions a physical device into mutually exclusive virtual devices (e.g. an NVIDIA GPU exposed as both MIG partitions and an MPS partition) cannot tell the scheduler that these exposures are hardware-exclusive. Under counter-only accounting, the scheduler may pick a combination that cannot actually be realized, causing thrashing: `NodePrepareResources` fails on the kubelet, the claim is torn down, the pod is rescheduled, the same bad combination may be chosen again on another node.

**Solution.** Devices declare membership in one or more *compatibility groups* per counter-set entry. Two devices sharing a counter set can be co-allocated only if they share **at least one** group name (set-intersection non-empty), or if **both** omit the field. The relation is symmetric but not transitive. No cluster-wide naming registry is introduced; drivers define their own group names and document them.

**Alpha target.** `v1.37`, behind feature gate `DRADeviceCompatibilityGroups` (default off). Components gated: `kube-apiserver` (strips the field on writes when disabled) and `kube-scheduler` (ignores the field / skips intersection checks when disabled).

**Status today.** Branch `kep-5963` is clean, equal to `master`. No code has been written yet.

---

## Approach

The feature is intentionally small-surface: one new optional slice field on an existing versioned type, plus an intersection check inside the existing DRA structured-allocator `allocateDevice` flow. No new CRDs, no API version bump, no CLI changes.

The KEP is alpha-scope in 1.37, so all behaviour goes behind a feature gate and the DRA scheduler plugin reads through the same `Features` struct already used for other alpha DRA features (`ListTypeAttributes`, `ConsumableCapacity`, etc.). The allocator change goes into the **experimental** variant only; `incubating` and `stable` are left untouched until the feature graduates (the existing router in `staging/src/k8s.io/dynamic-resource-allocation/structured/allocator.go:204-248` will automatically pick `experimental` whenever the new feature bit is set).

---

## File-by-file changes

### 1. API types (versioned + internal + draapi)

Add an optional `CompatibilityGroups []string` field to `DeviceCounterConsumption`. Only the **v1** API ships in 1.37 for this feature; v1beta1/v1beta2 were kept in sync for existing `consumesCounters` fields, so match that convention.

- **`staging/src/k8s.io/api/resource/v1/types.go:521-538`** — add field after `Counters`:
  ```go
  // CompatibilityGroups lists names of mutually-compatible groups this
  // device belongs to for the referenced counter set. Two devices
  // consuming from the same counter set can be co-allocated only if
  // they share at least one group name, or if both omit the field.
  //
  // The relation is symmetric but not transitive.
  // Group names are driver-defined; no cluster-wide registry is enforced.
  //
  // The maximum number of groups per entry is 8.
  //
  // +optional
  // +listType=atomic
  // +featureGate=DRADeviceCompatibilityGroups
  CompatibilityGroups []string `json:"compatibilityGroups,omitempty" protobuf:"bytes,3,rep,name=compatibilityGroups"`
  ```
  Use protobuf tag `3` (after `counterSet=1`, `counters=2`).

- **`staging/src/k8s.io/api/resource/v1beta1/types.go:523-540`** — mirror (keep versions in sync).
- **`staging/src/k8s.io/api/resource/v1beta2/types.go:506-523`** — mirror.
- **`pkg/apis/resource/types.go:482-495`** — add internal field (no tags):
  ```go
  CompatibilityGroups []string
  ```
- **`staging/src/k8s.io/dynamic-resource-allocation/api/types.go:78-81`** — add field to the scheduler-internal draapi mirror:
  ```go
  CompatibilityGroups []string `json:",omitempty"`
  ```

Add a constant `ResourceSliceMaxCompatibilityGroupsPerConsumption = 8` next to the existing `ResourceSliceMaxCountersPerDeviceCounterConsumption = 32` in each versioned package (the same spots listed at lines 502-503 of each v1*/types.go).

### 2. Feature gate

- **`pkg/features/kube_features.go:251`** — insert a new declaration alphabetically between `DRADeviceBindingConditions` and `DRADeviceTaintRules` (alphabetic / topical grouping):
  ```go
  // owner: @omeryahud
  // kep: https://kep.k8s.io/5963
  //
  // Enables DRA drivers to express mutual-compatibility constraints
  // between devices that consume the same counter set, so the scheduler
  // can avoid co-allocating incompatible partitioning modes.
  DRADeviceCompatibilityGroups featuregate.Feature = "DRADeviceCompatibilityGroups"
  ```
- **`pkg/features/versioned_kube_features.go`** (or whichever file holds `defaultKubernetesFeatureGates`) — add entry after the `DRAPartitionableDevices` block (`pkg/features/kube_features.go:1366-1369` shows the pattern):
  ```go
  DRADeviceCompatibilityGroups: {
      {Version: version.MustParse("1.37"), Default: false, PreRelease: featuregate.Alpha},
  },
  ```

### 3. Registry strategy — drop disabled fields

- **`pkg/registry/resource/resourceslice/strategy.go:193-201`** — add a new dispatch call:
  ```go
  dropDisabledDRADeviceCompatibilityGroupsFields(newSlice, oldSlice)
  ```
- Add alongside `dropDisabledDRAPartitionableDevicesFields` (strategy.go:226-238) following the same ratchet pattern:
  ```go
  func dropDisabledDRADeviceCompatibilityGroupsFields(newSlice, oldSlice *resource.ResourceSlice) {
      if utilfeature.DefaultFeatureGate.Enabled(features.DRADeviceCompatibilityGroups) ||
          draDeviceCompatibilityGroupsFeatureInUse(oldSlice) {
          return
      }
      for i := range newSlice.Spec.Devices {
          for j := range newSlice.Spec.Devices[i].ConsumesCounters {
              newSlice.Spec.Devices[i].ConsumesCounters[j].CompatibilityGroups = nil
          }
      }
  }

  func draDeviceCompatibilityGroupsFeatureInUse(slice *resource.ResourceSlice) bool {
      if slice == nil {
          return false
      }
      for _, device := range slice.Spec.Devices {
          for _, cc := range device.ConsumesCounters {
              if len(cc.CompatibilityGroups) > 0 {
                  return true
              }
          }
      }
      return false
  }
  ```

### 4. API defaulting / conversion

No default values (empty and nil are equivalent per the KEP). The existing auto-generated conversion machinery handles the new slice field correctly once codegen is re-run (`pkg/apis/resource/v1/conversion.go`, `pkg/apis/resource/v1beta1/conversion.go:198-267`, v1beta2 similarly) — no hand-written changes expected.

### 5. API validation

- **`pkg/apis/resource/validation/validation.go:948-963`** (`validateDeviceCounterConsumption`) — extend to validate the new field when non-empty. Add:
  ```go
  if len(deviceCounterConsumption.CompatibilityGroups) > 0 {
      allErrs = append(allErrs, validateSet(deviceCounterConsumption.CompatibilityGroups,
          resource.ResourceSliceMaxCompatibilityGroupsPerConsumption,
          validateCompatibilityGroupName,
          func(s string) (string, string) { return s, "" },
          fldPath.Child("compatibilityGroups"))...)
  }
  ```
  `validateCompatibilityGroupName` reuses `validateCounterName` semantics (DNS label, ≤63 chars). Place the helper next to existing `validateCounterName` in the same file.

### 6. Structured-allocator Features flag

- **`staging/src/k8s.io/dynamic-resource-allocation/structured/internal/features.go`** (the file that defines the `Features` struct — grep for `PartitionableDevices bool` to locate) — add:
  ```go
  DeviceCompatibilityGroups bool
  ```
  and include it in `FeaturesAll` used by the test harness.

- **`staging/src/k8s.io/dynamic-resource-allocation/structured/internal/experimental/allocator_experimental.go:77-84`** — extend `SupportedFeatures`:
  ```go
  var SupportedFeatures = internal.Features{
      ...
      DeviceCompatibilityGroups: true,
  }
  ```
  Leave `stable` and `incubating` unchanged — their `SupportedFeatures` must NOT set this bit, so the router at `structured/allocator.go:204-248` falls through to experimental whenever the feature is enabled. This matches the behaviour for `ListTypeAttributes`, which is experimental-only today.

### 7. Allocator logic — compatibility-group enforcement

All changes in **`staging/src/k8s.io/dynamic-resource-allocation/structured/internal/experimental/allocator_experimental.go`**.

- **`allocator` struct** (mirrors `alloc_stable.go:499-522`) — add a new per-attempt map tracking the groups already chosen per pool+counterSet:
  ```go
  // groupsPerCounterSet tracks, for each (pool, counterSet), the set of
  // compatibility-group names declared by devices being allocated in the
  // current attempt. Nil entry means "no constraints yet"; empty set
  // means "a device declared no groups" (locks out any grouped device).
  groupsPerCounterSet map[draapi.UniqueString]map[draapi.UniqueString]compatState
  ```
  where `compatState` is a tiny type:
  ```go
  type compatState struct {
      // groups is the intersection of compatibility groups declared by
      // allocating devices so far on this counter set. nil means "no
      // grouped device yet". empty (non-nil) means "an ungrouped device
      // was added — subsequent devices must also be ungrouped".
      groups sets.Set[string]
      // ungroupedSeen records whether at least one ungrouped device has
      // joined this counter set in the current attempt.
      ungroupedSeen bool
  }
  ```

- **`allocateDevice`** (lines 1454 ff., mirroring `stable/allocator_stable.go:1104-1208`) — after the existing counter-availability check at `experimental/allocator_experimental.go:1454-1475`, add a new guarded block:
  ```go
  if alloc.features.DeviceCompatibilityGroups && len(device.ConsumesCounters) > 0 {
      ok := alloc.checkCompatibilityGroups(device)
      if !ok {
          alloc.logger.V(7).Info("Device rejected by compatibility groups",
              "device", device.id)
          return false, nil, nil
      }
  }
  ```
  and extend the rollback closure (end of `allocateDevice`) to call `alloc.rollbackCompatibilityGroups(device)` alongside the existing `deallocateCountersForDevice`.

- **`checkAvailableCounters`** (`experimental/allocator_experimental.go:1526` ff., mirroring `stable/allocator_stable.go:1237-1346`) — when initializing the per-pool cache, *also* seed `groupsPerCounterSet` with the compatibility-group state derived from devices in `alloc.allocatedDevices` (already persisted allocations). Walk the same `slice.Spec.Devices` loop that builds counter usage at lines 1277-1290 of `stable`; for each already-allocated device, fold its `CompatibilityGroups` into the state. Unallocated devices are skipped.

- New helpers at the bottom of `allocator_experimental.go`:
  ```go
  // checkCompatibilityGroups returns true iff `device` is compatible with
  // the devices already being allocated on every counter set it consumes.
  // Rules (KEP-5963):
  //   - both ungrouped -> compatible
  //   - one ungrouped, one grouped -> incompatible
  //   - both grouped -> intersection must be non-empty
  // Also mutates the running `groupsPerCounterSet` state to include
  // `device` on success. Callers must call rollbackCompatibilityGroups
  // to undo on subsequent failure of the surrounding attempt.
  func (alloc *allocator) checkCompatibilityGroups(device deviceWithID) bool {
      poolName := device.pool.PoolID.Pool
      poolState, ok := alloc.groupsPerCounterSet[poolName]
      if !ok {
          poolState = make(map[draapi.UniqueString]compatState)
          alloc.groupsPerCounterSet[poolName] = poolState
      }
      for _, dcc := range device.ConsumesCounters {
          st := poolState[dcc.CounterSet]
          deviceHasGroups := len(dcc.CompatibilityGroups) > 0
          if st.ungroupedSeen && deviceHasGroups {
              return false
          }
          if !deviceHasGroups {
              if st.groups != nil {
                  return false
              }
              st.ungroupedSeen = true
              poolState[dcc.CounterSet] = st
              continue
          }
          deviceSet := sets.New(dcc.CompatibilityGroups...)
          if st.groups == nil {
              st.groups = deviceSet
          } else {
              st.groups = st.groups.Intersection(deviceSet)
              if st.groups.Len() == 0 {
                  return false
              }
          }
          poolState[dcc.CounterSet] = st
      }
      return true
  }
  ```
  and a symmetrical `rollbackCompatibilityGroups` that reverses the single device's contribution. The simplest correct rollback is to recompute from scratch for the touched counter sets (the attempt state is small), but an incremental reverse is also acceptable.

- **`NewAllocator`** in the experimental package — initialize `groupsPerCounterSet: make(...)` alongside `consumedCounters`.

### 8. Metrics (alpha-grade)

- **`pkg/scheduler/metrics/metrics.go`** (pattern at lines 470-489) — register:
  ```go
  DRACompatibilityGroupRejectionsTotal = metrics.NewCounterVec(
      &metrics.CounterOpts{
          Subsystem:      SchedulerSubsystem,
          Name:           "dra_compatibility_rejections_total",
          Help:           "Count of DRA candidate-device rejections due to compatibility-group conflicts, by driver and counter set.",
          StabilityLevel: metrics.ALPHA,
      },
      []string{"driver", "counter_set"},
  )
  ```
- Increment from the allocator via a thin callback or from the scheduler plugin after `ErrFailedAllocationOnNode` classification. Simplest path: increment inside `checkCompatibilityGroups` on the `return false` paths, guarded by `alloc.features.DeviceCompatibilityGroups`.

### 9. Regenerate derived artifacts

Run `./hack/update-codegen.sh` to regenerate:
- `zz_generated.deepcopy.go` under `staging/src/k8s.io/api/resource/{v1,v1beta1,v1beta2}` and `pkg/apis/resource`.
- `zz_generated.conversion.go` for resource/v1*.
- `generated.pb.go` protobuf marshalers for resource/v1*.
- `types_swagger_doc_generated.go`.
- `zz_generated.validations.go` if declarative validation picks up the new field.
- `api/openapi-spec/v3/apis__resource.k8s.io__v{1,1beta1,1beta2}_openapi.json`.
- `api/api-rules/violation_exceptions.list` may need no changes (slice of string).

Also update the API-diff golden file under `api/testdata/` that records new fields per version (hack script: `./hack/update-golangci-lint-config.sh` is unrelated; the right one is `./hack/update-openapi-spec.sh` already invoked by `update-codegen.sh`).

---

## Testing

### Unit tests

- **`pkg/registry/resource/resourceslice/strategy_test.go`** — extend `sliceWithPartitionableDevicesConsumesCounters` fixture (lines 97-138) with a version that has `CompatibilityGroups: []string{"mig"}` and add table entries verifying:
  - Field dropped on create when gate disabled and `oldSlice` didn't use the field.
  - Field preserved on update when `oldSlice` already had it (ratchet).
- **`pkg/apis/resource/validation/validation_resourceslice_test.go:1132-1160`** — add table cases:
  - Valid single group, multiple groups, empty list (treated as nil).
  - Invalid: group count over cap, invalid group name (uppercase, too long).
- **`staging/src/k8s.io/dynamic-resource-allocation/structured/internal/allocatortesting/allocator_testing.go`** — add test scenarios (this is the shared harness invoked by stable/incubating/experimental allocator unit tests):
  - Two devices sharing one group → allocate together.
  - Two devices with disjoint groups → second rejected.
  - Ungrouped + grouped in same counter set → rejected in either order.
  - Non-transitive: A∩B, B∩C, A∩C=∅ → A+C rejected.
  - Feature-gate off → field ignored.
- **`pkg/scheduler/framework/plugins/dynamicresources/dynamicresources_test.go`** — add a filter-level test that sets up two candidate devices with conflicting groups and asserts the node is filtered out, plus a metric-assertion using `testutil.ToFloat64`.

### Integration tests

- New file **`test/integration/dra/compatibility_groups.go`** modeled on `test/integration/dra/partitionable_devices.go:30-95`. Cases:
  - Feature disabled: creating a slice with `compatibilityGroups` strips the field.
  - Feature enabled: driver publishes two MIG devices (group `mig`) and one MPS device (group `mps`) consuming the same counter set; two pods can claim MIG, but a pod asking for MIG + MPS cannot be scheduled.
  - Feature enable → disable → enable (ratchet): downgrade preserves existing slices; re-upgrade doesn't lose data.

### E2E

Out of scope for alpha per the KEP. The fake DRA driver under `test/e2e/dra/test-driver/` will gain compatibility-groups support as a follow-up PR in the beta milestone (`v1.38`).

---

## Verification

1. `make all WHAT=./cmd/kube-apiserver ./cmd/kube-scheduler` — binaries compile.
2. `./hack/verify-codegen.sh` — generated code up to date.
3. `./hack/verify-openapi-spec.sh` — openapi diff accepted.
4. Run unit tests:
   ```
   make test WHAT=./pkg/apis/resource/validation ./pkg/registry/resource/resourceslice \
       ./staging/src/k8s.io/dynamic-resource-allocation/... \
       ./pkg/scheduler/framework/plugins/dynamicresources
   ```
5. Run integration tests:
   ```
   make test-integration WHAT=./test/integration/dra GOFLAGS='-v' \
       KUBE_TEST_ARGS='-run TestCompatibilityGroups'
   ```
6. Manual smoke (optional): start a local cluster with `local-up-cluster.sh`, enable `--feature-gates=DRADeviceCompatibilityGroups=true` on both apiserver and scheduler, and apply the YAML snippet from the KEP.

---

## Critical files (summary)

| Area | Path | Role |
|---|---|---|
| API | `staging/src/k8s.io/api/resource/v1/types.go` | add field (line ~537) |
| API | `staging/src/k8s.io/api/resource/v1beta1/types.go` | mirror |
| API | `staging/src/k8s.io/api/resource/v1beta2/types.go` | mirror |
| API | `pkg/apis/resource/types.go` | internal type (line ~495) |
| API | `staging/src/k8s.io/dynamic-resource-allocation/api/types.go` | draapi mirror (line ~81) |
| Gate | `pkg/features/kube_features.go` | declare + register gate |
| Strategy | `pkg/registry/resource/resourceslice/strategy.go` | `dropDisabledDRADeviceCompatibilityGroupsFields` + dispatch |
| Validation | `pkg/apis/resource/validation/validation.go` | extend `validateDeviceCounterConsumption` |
| Allocator | `staging/src/k8s.io/dynamic-resource-allocation/structured/internal/features.go` | add `DeviceCompatibilityGroups` bool |
| Allocator | `staging/src/k8s.io/dynamic-resource-allocation/structured/internal/experimental/allocator_experimental.go` | group-intersection check + state |
| Metrics | `pkg/scheduler/metrics/metrics.go` | `DRACompatibilityGroupRejectionsTotal` |
| Tests | `pkg/registry/resource/resourceslice/strategy_test.go` | strategy drops |
| Tests | `pkg/apis/resource/validation/validation_resourceslice_test.go` | validation |
| Tests | `staging/src/k8s.io/dynamic-resource-allocation/structured/internal/allocatortesting/allocator_testing.go` | allocator matrix |
| Tests | `test/integration/dra/compatibility_groups.go` (new) | integration |

## Out of scope for this PR

- Beta/GA graduation tasks (e2e tests, benchmarks, production-driver validation).
- Changes to `incubating` and `stable` allocator variants (feature will be added there on graduation).
- Any CLI / kubectl surface — the field is driver-authored, not user-facing.
- Cross-counter-set compatibility (explicit KEP non-goal).
- Cluster-wide group-name registry (explicit KEP non-goal).
