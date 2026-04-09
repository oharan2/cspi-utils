# Onboarding a layered-product interop component (`*-lp-interop`)

This document is a **checklist for coding agents** (and humans) adding support in Sippy for a new **MY-CMP** product that publishes CI under the layered-product / lp-interop pattern (JUnit suite like `MY-CMP-lp-interop`, Prow jobs under `…-lp-interop-…`).

Follow the steps in order. **Do not run any `make` commands** from automation; a maintainer must run them locally when noted below.

---

## 0. Prerequisites: gather required information

From `openshift/release` (or your team’s CI config), confirm:

1. **Mapped `testSuites` component name** as it appears in imported test data (often `ProductName-lp-interop`, e.g. `OADP-lp-interop`). It must match **exactly** (case-sensitive) what you add to `testSuites`.
2. **Stable substring of periodic name**, e.g. `-lp-interop-cr-<something>`. The variant registry matches **literal substrings** on the lowercased job name (first match wins).  
    * Whether you need **multiple** patterns (e.g. `-lp-interop-cr-acs` vs `-lp-interop-cr-acs-latest`)—add **separate** rows, **more specific before more general**.

---

## 1. Allow importing tests: `pkg/db/suites.go`

**File:** `pkg/db/suites.go`  
**List:** `testSuites`

- Append the **exact** mapped JUnit suite name, e.g. `MyProduct-lp-interop`.
- Keep the list **sorted in a sensible way** (group with other `*-lp-interop` suites).

Tests from suites **not** in this list are **not** imported into Sippy’s DB.

---

## 2. Map Prow job names → `LayeredProduct`: `pkg/variantregistry/ocp.go`

**File:** `pkg/variantregistry/ocp.go`  
**Function:** `setLayeredProduct`

Add an entry to `layeredProductPatterns`:

```go
{"-lp-interop-cr-my-cmp", "lp-interop-my-cmp"},
```

**Rules:**

- **`product` value:** always use the **`lp-interop-…`** form (lowercase, hyphenated), e.g. `lp-interop-my-cmp`. This is what Component Readiness views filter on.
- **`substring`:** must appear in real periodic job names after lowercasing. Align with CI naming (often `-lp-interop-cr-<repo-or-product-slug>`).
- **Order matters:** the slice is scanned **top to bottom**; the **first** match wins. Place **narrow** patterns (e.g. product-specific) **above** broad patterns like `{"-virt", "virt"}` so lp-interop jobs are not misclassified.

**Optional (IBM / on-prem style job names):** If jobs include `-ibm` / `-ibmcloud` and you want them bucketed with bare metal for platform filtering, confirm `setPlatform` includes the `{"-ibm", "metal"}` mapping (or add it if your branch does not). That is **independent** of lp-interop onboarding but affects which **Platform** filter includes those jobs.

---

## 3. Include the product in LP-Interop views: `config/views.yaml`

**File:** `config/views.yaml`

For each Component Readiness view named like **`*-LP-Interop`** (e.g. `4.22-LP-Interop`, `4.21-LP-Interop`) that lists layered products under:

```yaml
variant_options:
  include_variants:
    LayeredProduct:
      - lp-interop-...
```

add:

```yaml
      - lp-interop-my-cmp
```

Use the **same string** as in `setLayeredProduct`’s `product` field. Keep the list **alphabetically sorted** unless the file already uses a different convention for that block.

**Note:** Some older views (e.g. certain `4.20-*` LP views) may only list a subset of products—only add your entry where other `lp-interop-*` products are already listed.

---

## 4. Tests and variant snapshot (do not run `make` here)

### 4a. Unit test (recommended)

**File:** `pkg/variantregistry/ocp_test.go`  
**Test:** `TestVariantSyncer`

Add a case with a **realistic** periodic job name for MY-CMP (including release and network tokens if needed) and assert `VariantPlatform`, `VariantLayeredProduct`, etc., match what `IdentifyVariants` returns.

### 4b. Variant snapshot

**Test:** `TestVariantsSnapshot` in `pkg/variantregistry/ocp_test.go` compares live variants for all jobs in `config/openshift.yaml` against **`pkg/variantregistry/snapshot.yaml`**.

After **any** change to variant logic in `pkg/variantregistry/ocp.go` (including `setLayeredProduct` / `setPlatform`), that snapshot **must** be regenerated or the test will fail.

**Agents must not run `make`.** Ask the maintainer to run, **after** your Go changes are merged or applied locally:

```bash
make update-variants
```

That target builds `./sippy` and runs:

```bash
./sippy variants snapshot --config ./config/openshift.yaml
```

which rewrites `pkg/variantregistry/snapshot.yaml`.

---

## 5. Quick verification (optional; no `make`)

From the repo root (if the environment allows):

```bash
go test ./pkg/variantregistry/ -run 'TestVariantSyncer' -count=1
```

Expect snapshot tests to fail until `make update-variants` has been run by a maintainer.

---

## 6. What **not** to do

- Do **not** run **`make`** (including `make update-variants`, `make`, `make test`, etc.) from the agent; record **`make update-variants`** for the human when variant code changed.
- Do **not** hand-edit **`snapshot.yaml`** unless you have a documented, repo-approved process; prefer **`make update-variants`**.
- Do **not** change unrelated views, suites, or variant patterns.
- After frontend changes under `sippy-ng`, this onboarding path does not require npm; if you touch JS, follow `AGENTS.md` (eslint/prettier) separately.

---

## Summary checklist

| Step | Location | Action |
|------|----------|--------|
| 1 | `pkg/db/suites.go` | Add JUnit suite `MyProduct-lp-interop` to `testSuites` |
| 2 | `pkg/variantregistry/ocp.go` | Add `setLayeredProduct` pattern → `lp-interop-my-cmp` |
| 3 | `config/views.yaml` | Add `lp-interop-my-cmp` to `*-LP-Interop` views’ `LayeredProduct` |
| 4 | `pkg/variantregistry/ocp_test.go` | Add `TestVariantSyncer` case (recommended) |
| 5 | Maintainer | Run **`make update-variants`** after variant changes |

Replace `my-cmp` / `MyProduct` with your actual product slug and suite name throughout.
