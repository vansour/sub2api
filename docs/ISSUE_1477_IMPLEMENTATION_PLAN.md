# Issue #1477 Implementation Plan

Last updated: 2026-04-10

Related issues:
- #1477 OpenAI 使用compact压缩问题
- #805 Codex Compact Error
- #849 Error running remote compact task
- #661 是否考虑增加 responses/compact 支持

## Goal

Implement `/responses/compact` as a first-class OpenAI capability in Sub2API.

This plan assumes:
- `compact` is an endpoint capability, not a normal model.
- Runtime capability state should live in `accounts.extra`.
- Compact-only model remapping should live in `accounts.credentials`.
- No database migration is required for the first implementation.

## Current State

Already implemented in current codebase:
- `/v1/responses/*subpath` and `/responses/*subpath` route to OpenAI Responses handler.
- Compact path detection and compact body normalization exist.
- Upstream forwarding preserves `/compact` suffix.
- Compact requests already force JSON accept headers and session headers.

Missing today:
- Account-level compact capability modeling.
- Compact-aware account scheduling.
- Active compact probe during account test.
- Compact-only model remapping for accounts whose upstream only supports a special compact model.
- Admin UI for capability visibility and manual override.

## Design Principles

1. Do not treat compact as a normal model.
2. Do not overload existing `model_mapping` with compact-only semantics.
3. Do not mark an account unsupported based on transient upstream failures.
4. Prefer automatic capability discovery, but allow manual override.
5. Keep unknown capability states usable during rollout.

## Data Contract

### `accounts.extra`

Suggested keys:
- `openai_compact_mode`: `"auto" | "force_on" | "force_off"`
- `openai_compact_supported`: `bool`
- `openai_compact_checked_at`: RFC3339 string
- `openai_compact_last_status`: integer
- `openai_compact_last_error`: string

Semantics:
- `force_on`: always treat account as compact-capable
- `force_off`: never use account for compact requests
- `auto`: follow probe result
- missing `openai_compact_supported`: capability is unknown

### `accounts.credentials`

Suggested key:
- `compact_model_mapping`: `map[string]string`

Purpose:
- only applied for `/responses/compact`
- does not affect normal `/responses`

## Phase 1: Domain Helpers And Field Conventions

### Objective

Add explicit compact capability semantics without changing behavior yet.

### Backend files to change

- `backend/internal/service/account.go`

### Functions and constants to add

- Add compact mode constants:
  - `OpenAICompactModeAuto`
  - `OpenAICompactModeForceOn`
  - `OpenAICompactModeForceOff`
- Add helpers on `Account`:
  - `GetOpenAICompactMode() string`
  - `OpenAICompactSupportKnown() (supported bool, known bool)`
  - `AllowsOpenAICompact() bool`
  - `GetCompactModelMapping() map[string]string`
  - `ResolveCompactMappedModel(requestedModel string) (mapped string, matched bool)`

### Expected behavior

- `force_on` means supported regardless of probe data.
- `force_off` means unsupported regardless of probe data.
- `auto` + missing support flag means `unknown`.
- `compact_model_mapping` supports exact keys and wildcard keys the same way normal `model_mapping` does.

### Tests to add

- `backend/internal/service/account_openai_compact_test.go`

Coverage:
- mode parsing
- supported vs unknown
- force overrides
- compact mapping exact match
- compact mapping wildcard match
- missing compact mapping falls back to original model

### Done criteria

- Compact helper methods exist and are covered by unit tests.
- No runtime behavior changes outside tests.

## Phase 2: Compact-Aware Scheduling

### Objective

Ensure compact requests select accounts that can actually handle compact.

### Backend files to change

- `backend/internal/handler/openai_gateway_handler.go`
- `backend/internal/service/openai_account_scheduler.go`
- `backend/internal/service/openai_gateway_service.go`
- `backend/internal/service/openai_ws_forwarder.go`

### Main implementation steps

1. Extend scheduler request shape.
   - Add `RequireCompact bool` to `OpenAIAccountScheduleRequest`.

2. Thread compact intent from handler to scheduler.
   - In `OpenAIGatewayHandler.Responses`, derive `requireCompact` from compact path detection.
   - Pass it into `SelectAccountWithScheduler`.

3. Add compact-aware account eligibility checks.
   - Suggested helper:
     - `isOpenAIAccountEligibleForRequest(account *Account, requestedModel string, requireCompact bool) bool`

4. Apply eligibility checks in all relevant paths.
   - `defaultOpenAIAccountScheduler.selectBySessionHash`
   - `defaultOpenAIAccountScheduler.selectByLoadBalance`
   - `OpenAIGatewayService.resolveFreshSchedulableOpenAIAccount`
   - `OpenAIGatewayService.recheckSelectedOpenAIAccountFromDB`
   - `OpenAIGatewayService.tryStickySessionHit`
   - `OpenAIGatewayService.selectBestAccount`
   - `OpenAIGatewayService.selectAccountForModelWithExclusions`
   - `OpenAIGatewayService.SelectAccountByPreviousResponseID`

5. Implement capability tiers.
   - Tier 1: explicit supported
   - Tier 2: unknown
   - Tier 3: explicit unsupported

Selection rule:
- prefer supported
- fallback to unknown
- reject unsupported

### Error handling

When all remaining accounts are explicitly unsupported for compact:
- return a clear compact-specific error
- avoid generic `Service temporarily unavailable`

Suggested error code/message:
- code: `compact_not_supported`
- message: `No available OpenAI accounts support /responses/compact`

### Tests to update/add

- Update:
  - `backend/internal/service/openai_account_scheduler_test.go`
  - `backend/internal/service/openai_ws_account_sticky_test.go`
- Add if useful:
  - `backend/internal/service/openai_account_scheduler_compact_test.go`

Coverage:
- compact requests prefer supported accounts
- compact requests can fallback to unknown
- compact requests never choose explicitly unsupported accounts
- sticky session is cleared when sticky account becomes compact-ineligible
- `previous_response_id` sticky lookup rejects compact-ineligible accounts
- non-compact requests keep old behavior

### Done criteria

- Compact requests no longer reach explicitly unsupported accounts.
- Existing non-compact scheduling tests still pass.

## Phase 3: Compact-Only Model Mapping

### Objective

Support accounts whose upstream requires a special compact model, such as a compact-specific OpenAI model.

### Backend files to change

- New file:
  - `backend/internal/service/openai_compact_model_mapping.go`
- Update:
  - `backend/internal/service/openai_gateway_service.go`

### Main implementation steps

1. Add helper:
   - `resolveOpenAICompactForwardModel(account *Account, requestedModel string) string`

2. Insert compact-only mapping into the OpenAI request rewrite flow.

Recommended order inside OpenAI forwarding:
1. apply normal `model_mapping`
2. if compact path, apply `compact_model_mapping`
3. apply OAuth upstream normalization

Relevant code area:
- `OpenAIGatewayService.Forward`
- model rewrite block around existing normal model mapping and `applyCodexOAuthTransform`

### Important rule

Do not reuse normal `model_mapping` for compact-only remapping.

Reason:
- normal `/responses` and compact `/responses/compact` must remain independent

### Tests to add/update

- Add:
  - `backend/internal/service/openai_compact_model_mapping_test.go`
- Update:
  - `backend/internal/service/openai_gateway_service_test.go`

Coverage:
- compact request uses `compact_model_mapping`
- normal request ignores `compact_model_mapping`
- missing compact mapping keeps current behavior
- OAuth and API key variants both covered

### Done criteria

- Compact-only remapping works without affecting normal requests.

## Phase 4: Active Compact Probe And Persistence

### Objective

Actively discover compact support during account test instead of relying only on production traffic.

### Backend files to change

- `backend/internal/handler/admin/account_handler.go`
- `backend/internal/service/account_test_service.go`
- New file:
  - `backend/internal/service/openai_compact_probe.go`
- Possibly update:
  - `backend/internal/repository/account_repo.go`

### API contract change

Extend admin account test request:
- existing fields:
  - `model_id`
  - `prompt`
- add:
  - `mode: "default" | "compact"`

### Main implementation steps

1. Extend `TestAccountRequest` in admin handler.
2. Extend `AccountTestService.TestAccountConnection(...)` to accept test mode.
3. For OpenAI account tests:
   - `mode=default`: keep existing behavior
   - `mode=compact`: perform compact probe

### Compact probe request behavior

OAuth:
- target `https://chatgpt.com/backend-api/codex/responses/compact`

API key:
- target `{baseURL}/responses/compact`

Request shape:
- minimal compact-valid body
- compact headers
- JSON accept
- session id

### Probe result classification

Persist:
- success: supported=true
- explicit endpoint/capability failure such as 404 or known unsupported 4xx: supported=false
- 5xx/network/timeout: do not set supported=false

Always persist diagnostics when available:
- `openai_compact_checked_at`
- `openai_compact_last_status`
- `openai_compact_last_error`

### Scheduler cache impact

Compact capability keys are scheduler-relevant.

Do not classify them as scheduler-neutral in:
- `schedulerNeutralExtraKeys`
- `schedulerNeutralExtraKeyPrefixes`

If capability flips, scheduler must see it promptly.

### Tests to add/update

- Add:
  - `backend/internal/service/account_test_service_openai_compact_test.go`
- Update:
  - admin handler tests around account test endpoint
  - `backend/internal/repository/account_repo_integration_test.go`

Coverage:
- compact test success writes support=true
- compact 404 writes support=false
- compact 502 only writes diagnostics, not support=false
- relevant `UpdateExtra` keys trigger scheduler outbox rather than neutral snapshot sync

### Done criteria

- Admin can actively test compact support.
- Probe results influence scheduling after persistence.

## Phase 5: Admin UI And Types

### Objective

Expose compact capability state and manual override in the admin console.

### Frontend files to change

- `frontend/src/types/index.ts`
- `frontend/src/components/account/CreateAccountModal.vue`
- `frontend/src/components/account/EditAccountModal.vue`
- `frontend/src/components/account/AccountTestModal.vue`
- `frontend/src/views/admin/AccountsView.vue`
- `frontend/src/i18n/locales/zh.ts`
- `frontend/src/i18n/locales/en.ts`

### Type updates

Extend account-facing types with:
- `openai_compact_mode`
- `openai_compact_supported`
- `openai_compact_checked_at`
- `openai_compact_last_status`
- `openai_compact_last_error`

Document `credentials.compact_model_mapping`.

### UI updates

In OpenAI account create/edit forms:
- add compact mode selector:
  - Auto
  - Force On
  - Force Off
- add compact-only model mapping editor

In account test modal:
- add test mode selector:
  - Default test
  - Compact test

In accounts list:
- optionally add compact support badge:
  - supported
  - unknown
  - unsupported

### Frontend tests to add/update

- Update:
  - `frontend/src/components/account/__tests__/EditAccountModal.spec.ts`
- Add:
  - `frontend/src/components/account/__tests__/CreateAccountModal.spec.ts`
  - `frontend/src/components/account/__tests__/AccountTestModal.spec.ts`

Coverage:
- compact mode writes correct `extra`
- compact model mapping writes correct `credentials.compact_model_mapping`
- compact test request posts `mode=compact`
- false values are not silently dropped on edit

### Done criteria

- Admin can view, override, and test compact support from UI.

## Recommended Delivery Order

1. Phase 1: domain helpers
2. Phase 2: compact-aware scheduling
3. Phase 4: active compact probe
4. Phase 3: compact-only model mapping
5. Phase 5: admin UI

Reason:
- Phase 2 immediately reduces bad account selection.
- Phase 4 turns unknown accounts into known states over time.
- Phase 3 solves compact-only upstream model requirements.
- Phase 5 is useful but not required to close the backend gap.

## Suggested PR Split

### PR 1

Title:
- `feat(openai): add compact capability helpers and scheduling filter`

Includes:
- Phase 1
- Phase 2

### PR 2

Title:
- `feat(openai): add compact probe to account test flow`

Includes:
- Phase 4

### PR 3

Title:
- `feat(openai): support compact-only account model mapping`

Includes:
- Phase 3

### PR 4

Title:
- `feat(admin): expose openai compact controls in account UI`

Includes:
- Phase 5

## Explicit Non-Goals For Initial Rollout

- No schema migration
- No group-level compact capability policy
- No channel-level compact policy
- No automatic production-failure-based permanent unsupported marking

## Final Success Criteria

The implementation is complete when all of the following are true:

- `/responses/compact` does not route to accounts explicitly known to be unsupported.
- Unknown accounts remain usable during rollout.
- Admin account test can actively probe compact support.
- Compact-only model remapping affects only compact requests.
- UI exposes compact status and override controls.
- Existing non-compact OpenAI flows remain unchanged.
