# Omni Official Store

## Current State
The app has an admin panel protected by a username/password login (omni_admin / Omni@2024) followed by Internet Identity login. After both logins, `upgradeToAdmin` is called once to register the principal as admin in the backend AccessControl state. However, admin operations (`generateRedeemCode`, `setSiteConfig`, `getAllOrders`, etc.) keep failing with "Unauthorized: Only admins can..." errors because:

1. `storeLocalParameter` is called in AdminPage.tsx but this function does NOT exist in urlParams.ts (only `storeSessionParameter` exists) -- so the token is never saved to localStorage
2. The backend `upgradeToAdmin` call only runs ONCE after login. Backend in-memory state can reset, so on subsequent uses the role may be gone
3. Admin API hooks (`useGetAllOrders`, `useGenerateRedeemCode`, `useSetSiteConfig`, etc.) call the backend WITHOUT first calling `upgradeToAdmin` -- so by the time they call the backend, the principal may not have admin role

## Requested Changes (Diff)

### Add
- `ADMIN_TOKEN` constant exported from a new `adminAuth.ts` utility file storing the Caffeine admin token
- A `getAdminToken()` helper in `adminAuth.ts` that returns the stored token from localStorage
- A `ensureAdminRole(actor)` async helper that always calls `upgradeToAdmin` before any admin action
- Logic in every admin query/mutation hook to call `ensureAdminRole` before the actual backend call

### Modify
- Fix `storeLocalParameter` calls in `AdminPage.tsx` -- replace with `localStorage.setItem("caffeineAdminToken", CAFFEINE_ADMIN_TOKEN)` since `storeLocalParameter` doesn't exist
- Update `AdminPage.tsx` to remove Internet Identity requirement entirely -- admin panel should work with just username/password + anonymous/any actor. The `upgradeToAdmin` call with the correct token is sufficient for backend auth.
- Update ALL admin hooks in `useQueries.ts` to call `actor.upgradeToAdmin(CAFFEINE_ADMIN_TOKEN)` immediately before every admin backend call (getAllOrders, generateRedeemCode, getAllRedeemCodes, setSiteConfig, setPaymentConfig, getAllTopUpRequests, approveTopUpRequest, rejectTopUpRequest, getUserStats, addPackage, updatePackage, removePackage, creditWallet, addGame, updateGame, removeGame, addBanner, updateBanner, removeBanner)
- The `AdminPage.tsx` `adminAuthed` check should be `passwordVerified` only (no `&&identity` requirement) -- Internet Identity is NOT required for admin panel to work since `upgradeToAdmin` handles the backend auth

### Remove
- The requirement for Internet Identity login in admin panel (the "Connect Internet Identity" second step screen)
- The `waitingForII`, `adminUpgradeDone` state and related useEffects in AdminPage that manage II login flow
- The "Connect Internet Identity" waiting screen

## Implementation Plan
1. In `AdminPage.tsx`: 
   - Export `CAFFEINE_ADMIN_TOKEN` constant (keep the existing value)
   - Fix `storeLocalParameter` → `localStorage.setItem` 
   - Change `adminAuthed = passwordVerified` (remove `&& !!identity`)
   - Remove II dependency from admin panel: the panel should load immediately after password verification without needing II
   - Remove the "waiting for II" screen entirely
   - Keep `useActor()` usage so backend calls work with anonymous principal, but always call `upgradeToAdmin` before admin ops
   
2. In `useQueries.ts`:
   - Import `CAFFEINE_ADMIN_TOKEN` from AdminPage or create a shared constant
   - Add `ensureAdmin` async helper: `async (actor) => { try { await actor.upgradeToAdmin(CAFFEINE_ADMIN_TOKEN) } catch {} }`
   - Call `ensureAdmin(actor)` at the start of EVERY admin hook's mutationFn/queryFn before the actual backend call
   - This guarantees the actor's principal has admin role for every single operation

3. Create `src/frontend/src/utils/adminAuth.ts`:
   - Export `CAFFEINE_ADMIN_TOKEN = "377fe7b083febffb7257d67a8c154bad9645538e0995c97c99df493c63c7be68"`
   - Export `ensureAdmin(actor)` helper function
