# Onboarding: layered-product interop components (`*-lp-interop`)

This guide describes how to add a new **layered product interop** component to [openshift-eng/ci-test-mapping](https://github.com/openshift-eng/ci-test-mapping).

Component Readiness maps each test to one **component** and optional **capabilities**. LP interop jobs publish JUnit with a dedicated mapped **test suite** name (for example `MyProduct-lp-interop`).

Replace placeholders below:

| Placeholder | Meaning |
|-------------|---------|
| `MyProduct-lp-interop` | Exact JUnit **suite** name from your interop tests (must match CI output). |
| `myproductlpinterop` | Go **package** name: lower case, no hyphens (typical pattern: strip `-lp-interop` and join words). |
| `MyProductLpInteropComponent` | Exported Go **variable** holding your component singleton. |

---

## Prerequisites

1. **Suite name is stable** and appears on every relevant JUnit result as the suite (same string your jobs already use, for example `MyProduct-lp-interop`).
2. **Registered `OCPBUGS` component name**: `DefaultJiraComponent` must correspond to a real Jira component the team owns.
   - Verify components with `./ci-test-mapping jira-verify` as described in the root [README.md](../../README.md#updating-jira-components).
   - Registered components can be found at [OCPBUGS components](https://redhat.atlassian.net/jira/software/c/projects/OCPBUGS/components).
---

## Note for AI / automation assistants

Do **not** run any `make` targets (or substitute commands) in this repository on behalf of the user. After editing `config/openshift-eng.yaml` or component code, **mapping regeneration is required** before merge: the human must run **`make mapping`** (see **Updating Mappings** in the root [README.md](../../README.md#updating-mappings)). State that requirement explicitly; do not execute it yourself. The human should review the resulting `data/` diff before opening a PR.

---

## 1. Include the suite in the OpenShift mapping config

Edit [config/openshift-eng.yaml](../../config/openshift-eng.yaml) and add your suite to `includeSuites`, in alphabetical order with the other `*-lp-interop` entries:

```yaml
includeSuites:
  # ... existing entries ...
  - MyProduct-lp-interop
```

Without this, tests from that suite may not appear in the mapping inputs at all.

---

## 2. Add a component package

Create a new directory:

`pkg/components/myproductlpinterop/`

### `component.go`

Model it on [pkg/components/fusionaccesslpinterop/component.go](../../pkg/components/fusionaccesslpinterop/component.go):

- Set `Name` and `DefaultJiraComponent` to your product’s Jira component name (often aligned with the suite, e.g. `MyProduct-lp-interop`).
- Use a matcher that claims **all tests in your suite**:

  ```go
  Matchers: []config.ComponentMatcher{{Suite: "MyProduct-lp-interop"}},
  ```

- Implement `IdentifyTest`, `StableID`, and `JiraComponents` the same way as the fusion-access example: return ownership when `FindMatch` hits; use `TestRenames` in `StableID` if tests are renamed; list Jira components in `JiraComponents`.

If you need finer-grained ownership later, add more `ComponentMatcher` entries (substrings, priorities, per-matcher Jira components) using patterns from [pkg/components/example](../../pkg/components/example).

### `capabilities.go`

Add a `capabilities.go` next to `component.go`, modeled on [pkg/components/fusionaccesslpinterop/capabilities.go](../../pkg/components/fusionaccesslpinterop/capabilities.go). It should define `identifyCapabilities` and start from `util.DefaultCapabilities(test)`; extend the returned slice only when you need capabilities beyond the defaults.

```go
package myproductlpinterop

import (
	v1 "github.com/openshift-eng/ci-test-mapping/pkg/api/types/v1"
	"github.com/openshift-eng/ci-test-mapping/pkg/util"
)

func identifyCapabilities(test *v1.TestInfo) []string {
	capabilities := util.DefaultCapabilities(test)
	return capabilities
}
```

---

## 3. Register the component

Edit [pkg/registry/registry.go](../../pkg/registry/registry.go):

1. Add the import:

   ```go
   "github.com/openshift-eng/ci-test-mapping/pkg/components/myproductlpinterop"
   ```

2. Register next to the other `*-lp-interop` components (keep ordering consistent with nearby entries):

   ```go
   r.Register("MyProduct-lp-interop", &myproductlpinterop.MyProductLpInteropComponent)
   ```

The string passed to `Register` is the **component name** used in mappings; it should match `Name` in your `config.Component` and is conventionally the same as the suite for LP interop.

---

## 4. Validate and ship

1. Regenerate committed mapping data: after changing config or components, **you must run** `make mapping` (see **Updating Mappings** in the root [README.md](../../README.md#updating-mappings)). 
2. AI/automation assistants must not run `make` for you; they should only flag that this step is required.

---

## Quick checklist

- [ ] `includeSuites` in `config/openshift-eng.yaml`
- [ ] `pkg/components/<package>/component.go` with `Suite: "MyProduct-lp-interop"`
- [ ] `pkg/components/<package>/capabilities.go` (`identifyCapabilities` + `util.DefaultCapabilities`, as in [fusionaccesslpinterop/capabilities.go](../../pkg/components/fusionaccesslpinterop/capabilities.go))
- [ ] Import + `r.Register(...)` in `pkg/registry/registry.go`
- [ ] Jira component exists / `jira-verify` clean
- [ ] Run **`make mapping`** (required)
