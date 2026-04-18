---
date: 2026-04-13T22:00:04+02:00
researcher: GitHub Copilot
git_commit: 6fc35fa05ec4f9f8cb9f993a01cda86d9a8cd8e0
branch: remove-enterprise-checks
repository: twenty
topic: "InformationBannerLegacyEnterpriseKey — validation flow and message delivery"
tags: [research, codebase, enterprise, information-banner, graphql, jotai]
status: complete
last_updated: 2026-04-13
last_updated_by: GitHub Copilot
---

# Research: InformationBannerLegacyEnterpriseKey — Validation and Message Display

**Date**: 2026-04-13T22:00:04+02:00
**Researcher**: GitHub Copilot
**Git Commit**: `6fc35fa05ec4f9f8cb9f993a01cda86d9a8cd8e0`
**Branch**: `remove-enterprise-checks`
**Repository**: twenty

## Research Question

Where is `InformationBannerLegacyEnterpriseKey` validated, and how is the message sent to the user?

## Summary

`InformationBannerLegacyEnterpriseKey` displays a deprecation banner when a workspace has a legacy (unsigned) enterprise key. The validation happens on two levels: first on the **backend** via NestJS `@ResolveField` resolvers that call `EnterprisePlanService`, and then on the **frontend** by reading the resolved boolean fields from Jotai state. The banner message is rendered inside a `<Banner>` component inside `InformationBannerWrapper`, which is mounted once at the app router level.

---

## Detailed Findings

### 1. Backend: GraphQL field resolution

**File**: [packages/twenty-server/src/engine/core-modules/workspace/workspace.resolver.ts](packages/twenty-server/src/engine/core-modules/workspace/workspace.resolver.ts)

Two `@ResolveField(() => Boolean)` resolvers are defined on `WorkspaceEntity`:

- **Line 298–302** — `hasValidEnterpriseKey()` → delegates to `this.enterprisePlanService.hasValidEnterpriseKey()`
- **Line 304–307** — `hasValidSignedEnterpriseKey()` → delegates to `this.enterprisePlanService.hasValidSignedEnterpriseKey()`

`EnterprisePlanService` is injected at line 101.

---

### 2. Backend: `EnterprisePlanService` validation logic

**File**: [packages/twenty-server/src/engine/core-modules/enterprise/services/enterprise-plan.service.ts](packages/twenty-server/src/engine/core-modules/enterprise/services/enterprise-plan.service.ts)

The service implements `OnModuleInit`. On startup (`onModuleInit`), it:
1. Calls `refreshKeyPayload()` — reads `ENTERPRISE_KEY` env var, attempts RS256 JWT verification, caches result as `cachedKeyPayload`.
2. Calls `loadValidityToken()` — loads an `EnterpriseValidityToken` from the `app_token` DB table (falling back to `ENTERPRISE_VALIDITY_TOKEN` env var), caches as `cachedValidityPayload`.

**`hasValidSignedEnterpriseKey()` (lines 130–133):**
```ts
hasValidSignedEnterpriseKey(): boolean {
  this.refreshKeyPayload();
  return isDefined(this.cachedKeyPayload);
}
```
Returns `true` only if `ENTERPRISE_KEY` is a cryptographically valid RS256 JWT.

**`hasValidEnterpriseKey()` (lines 146–157):**
```ts
hasValidEnterpriseKey(): boolean {
  return true; // temporary during migration period
  // intended logic (commented out):
  // if (this.hasValidSignedEnterpriseKey()) return true;
  // return this.checkLegacyKey();
}
```
**Currently hardcoded to `true`** during a migration period. The intended logic would check for a signed key first, then fall back to `checkLegacyKey()`.

**`checkLegacyKey()` (lines ~168–179):**
Accepts any non-empty `ENTERPRISE_KEY` env var without signature verification. Logs a deprecation warning and returns `true`.

**JWT signature verification** (`verifyJwt<T>()`, lines ~494–542):
- Splits JWT into header/payload/signature.
- Verifies with `crypto.verify()` using `RSA_PKCS1_PADDING`.
- In `NODE_ENV=development`, tries both `ENTERPRISE_JWT_PUBLIC_KEY` and `ENTERPRISE_JWT_DEV_PUBLIC_KEY`; in production, only `ENTERPRISE_JWT_PUBLIC_KEY`.
- Public keys are hardcoded at [packages/twenty-server/src/engine/core-modules/enterprise/constants/enterprise-public-key.constant.ts](packages/twenty-server/src/engine/core-modules/enterprise/constants/enterprise-public-key.constant.ts).

---

### 3. Backend: Type definitions

**File**: [packages/twenty-server/src/engine/core-modules/enterprise/types/enterprise-key-payload.type.ts](packages/twenty-server/src/engine/core-modules/enterprise/types/enterprise-key-payload.type.ts)

| Type | Fields |
|---|---|
| `EnterpriseKeyPayload` (line 1) | `sub`, `licensee`, `iat` — JWT stored in `ENTERPRISE_KEY` |
| `EnterpriseValidityPayload` (line 7) | `sub`, `status: 'valid'`, `iat`, `exp` — online validity token |
| `EnterpriseLicenseInfo` (line 14) | `isValid`, `licensee`, `expiresAt`, `subscriptionId` |

---

### 4. Frontend: GraphQL query that fetches the fields

**Query file**: [packages/twenty-front/src/modules/users/graphql/queries/getCurrentUser.ts](packages/twenty-front/src/modules/users/graphql/queries/getCurrentUser.ts)
**Fragment file**: [packages/twenty-front/src/modules/users/graphql/fragments/userQueryFragment.ts](packages/twenty-front/src/modules/users/graphql/fragments/userQueryFragment.ts)

`GetCurrentUser` uses `UserQueryFragment`, which selects `currentWorkspace { ... }` including (lines 63–64):

```graphql
hasValidEnterpriseKey
hasValidSignedEnterpriseKey
```

Both are `Boolean` scalars on the generated `Workspace` GraphQL type ([packages/twenty-front/src/generated-metadata/graphql.ts](packages/twenty-front/src/generated-metadata/graphql.ts#L6215-L6217)).

---

### 5. Frontend: Populating `currentWorkspaceState`

**Atom definition**: [packages/twenty-front/src/modules/auth/states/currentWorkspaceState.ts](packages/twenty-front/src/modules/auth/states/currentWorkspaceState.ts)
Created via `createAtomState<CurrentWorkspace | null>` with `defaultValue: null`. The `CurrentWorkspace` TypeScript type picks both `hasValidEnterpriseKey` and `hasValidSignedEnterpriseKey` from the `Workspace` type.

The atom is set in two places:

**A. `UserMetadataProviderInitialEffect`** ([packages/twenty-front/src/modules/metadata-store/effect-components/UserMetadataProviderInitialEffect.tsx](packages/twenty-front/src/modules/metadata-store/effect-components/UserMetadataProviderInitialEffect.tsx#L29)):
Uses a reactive Apollo `useQuery(GetCurrentUserDocument)`. When the query resolves, calls `setCurrentWorkspace(userQueryData.currentUser.currentWorkspace)` (lines ~95–103).

**B. `useLoadCurrentUser`** ([packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts](packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts#L49)):
Imperatively calls `client.query({ query: GetCurrentUserDocument, fetchPolicy: 'network-only' })`, then calls `setCurrentWorkspace(workspace)` (line 111).

---

### 6. Frontend: Banner component rendering

**Component**: [packages/twenty-front/src/modules/information-banner/components/InformationBannerLegacyEnterpriseKey.tsx](packages/twenty-front/src/modules/information-banner/components/InformationBannerLegacyEnterpriseKey.tsx)

The component reads `currentWorkspace` via `useAtomStateValue(currentWorkspaceState)` and computes:
```ts
const hasLegacyKey =
  currentWorkspace?.hasValidEnterpriseKey === true &&
  currentWorkspace?.hasValidSignedEnterpriseKey !== true;
```
If `hasLegacyKey` is `false`, returns `null`.

If `hasLegacyKey` is `true`, renders `<InformationBanner>` with:
- `componentInstanceId`: `'information-banner-legacy-enterprise-key'`
- `variant`: `'secondary'`
- `message`: `'Your enterprise key format is deprecated. Please activate a new key to keep enterprise features.'`
- `buttonTitle`: `'Activate'`
- `buttonIcon`: `IconKey`
- `buttonOnClick`: navigates to `getSettingsPath(SettingsPath.AdminPanelEnterprise)`
- `onClose`: sets `informationBannerIsOpenComponentState` to `false` for this instance

---

### 7. Frontend: `InformationBanner` component rendering

**File**: [packages/twenty-front/src/modules/information-banner/components/InformationBanner.tsx](packages/twenty-front/src/modules/information-banner/components/InformationBanner.tsx)

The component:
1. Reads `informationBannerIsOpen` via `useAtomComponentStateValue(informationBannerIsOpenComponentState, componentInstanceId)`.
2. Wraps output in `InformationBannerComponentInstanceContext.Provider`.
3. Returns `null` if `informationBannerIsOpen === false`.
4. Renders a `<Banner>` containing `<StyledContent>` with the message text.
5. If `buttonTitle` + `buttonOnClick` exist, renders a `<Button>` inside the banner.
6. If `onClose` is provided, renders a close icon button (variant depends on `variant` prop: `StyledInvertedIconButton` for `'primary'`, `IconButton` for `'secondary'`).

---

### 8. Frontend: Banner open/close state

**State**: [packages/twenty-front/src/modules/information-banner/states/informationBannerIsOpenComponentState.ts](packages/twenty-front/src/modules/information-banner/states/informationBannerIsOpenComponentState.ts)

```ts
export const informationBannerIsOpenComponentState =
  createAtomComponentState<boolean>({
    key: 'informationBannerIsOpenComponentState',
    defaultValue: true,  // banners start open by default
    componentInstanceContext: InformationBannerComponentInstanceContext,
  });
```

`useSetAtomComponentState` ([packages/twenty-front/src/modules/ui/utilities/state/jotai/hooks/useSetAtomComponentState.ts](packages/twenty-front/src/modules/ui/utilities/state/jotai/hooks/useSetAtomComponentState.ts)) resolves the instance via `instanceIdFromProps` or context, then returns `useSetAtom(componentState.atomFamily({ instanceId }))` — a Jotai atom setter scoped to that instance ID.

---

### 9. Frontend: Where `InformationBannerWrapper` is mounted

**Wrapper file**: [packages/twenty-front/src/modules/information-banner/components/InformationBannerWrapper.tsx](packages/twenty-front/src/modules/information-banner/components/InformationBannerWrapper.tsx)

`<InformationBannerLegacyEnterpriseKey />` is rendered unconditionally at line 58 (the component itself guards with `hasLegacyKey`). The wrapper is mounted inside `AppRouterProviders` (line 46 of [packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx](packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx#L46)).

---

## Code References

| Path | Description |
|---|---|
| `packages/twenty-server/src/engine/core-modules/workspace/workspace.resolver.ts:298–307` | `@ResolveField` resolvers for `hasValidEnterpriseKey` + `hasValidSignedEnterpriseKey` |
| `packages/twenty-server/src/engine/core-modules/enterprise/services/enterprise-plan.service.ts:130–157` | Service methods validating keys |
| `packages/twenty-server/src/engine/core-modules/enterprise/constants/enterprise-public-key.constant.ts` | Hardcoded RS256 public keys used for JWT verification |
| `packages/twenty-server/src/engine/core-modules/enterprise/types/enterprise-key-payload.type.ts` | Enterprise key TypeScript types |
| `packages/twenty-front/src/modules/users/graphql/fragments/userQueryFragment.ts:63–64` | GraphQL fragment selecting both enterprise key fields |
| `packages/twenty-front/src/modules/auth/states/currentWorkspaceState.ts` | Jotai atom storing current workspace including enterprise key fields |
| `packages/twenty-front/src/modules/metadata-store/effect-components/UserMetadataProviderInitialEffect.tsx:29` | Reactive query that populates `currentWorkspaceState` |
| `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:49,111` | Imperative query + atom setter for `currentWorkspaceState` |
| `packages/twenty-front/src/modules/information-banner/components/InformationBannerLegacyEnterpriseKey.tsx` | Banner component with `hasLegacyKey` guard |
| `packages/twenty-front/src/modules/information-banner/components/InformationBanner.tsx` | Base banner rendering component |
| `packages/twenty-front/src/modules/information-banner/states/informationBannerIsOpenComponentState.ts` | Jotai component atom controlling banner visibility |
| `packages/twenty-front/src/modules/information-banner/components/InformationBannerWrapper.tsx:58` | Unconditional inclusion of the legacy key banner |
| `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:46` | Mount point for `InformationBannerWrapper` |

---

## Architecture Documentation

### End-to-end flow

```
App startup
  └── UserMetadataProviderInitialEffect / useLoadCurrentUser
        └── Apollo GetCurrentUser query
              └── Backend: WorkspaceResolver.currentUser()
                    └── @ResolveField hasValidEnterpriseKey
                          └── EnterprisePlanService.hasValidEnterpriseKey()
                                → currently returns true (hardcoded)
                    └── @ResolveField hasValidSignedEnterpriseKey
                          └── EnterprisePlanService.hasValidSignedEnterpriseKey()
                                → isDefined(cachedKeyPayload)
                                  [RS256 JWT verification against hardcoded public keys]
  └── setCurrentWorkspace(currentUser.currentWorkspace)
        → Jotai atom: currentWorkspaceState

React render tree
  └── AppRouterProviders
        └── InformationBannerWrapper
              └── InformationBannerLegacyEnterpriseKey
                    → reads currentWorkspaceState
                    → computes hasLegacyKey =
                        hasValidEnterpriseKey === true
                        && hasValidSignedEnterpriseKey !== true
                    → if hasLegacyKey: renders InformationBanner
                          → reads informationBannerIsOpenComponentState (default: true)
                          → renders Banner with deprecation message
                          → "Activate" button → navigate to AdminPanelEnterprise
                          → "X" close → setInformationBannerIsOpen(false)
```

### Key architectural patterns

- **Two-layer guard**: The backend resolves whether keys are valid; the frontend additionally checks the combination `hasValidEnterpriseKey && !hasValidSignedEnterpriseKey` to identify the "legacy" state.
- **Jotai component atoms**: Banner visibility is stored in `informationBannerIsOpenComponentState` scoped by `instanceId` (`'information-banner-legacy-enterprise-key'`), so each banner manages its own open/close state independently.
- **Unconditional mount with internal guard**: `InformationBannerLegacyEnterpriseKey` is always mounted in the wrapper but returns `null` when the condition doesn't apply.
- **RS256 JWT signed keys**: The "new" enterprise key format is a JWT signed with a Twenty-controlled private key. The public keys for verification are hardcoded on the server.
