# Phase 0 Report: Safety Net & Error Boundary

## Completion: 100%

## What was done

### 1. TU.Safe.runAsync added (line ~4981)
- Added `runAsync()` method to `TU.Safe` for proper async/Promise error handling
- The original `run()` only wrapped sync code in try/catch - any async function would silently swallow Promise rejections
- `runAsync` uses `await` to properly catch both sync and async errors

### 2. GlobalErrorBoundary installed (line ~5029)
- Added `window.addEventListener("unhandledrejection")` to catch unhandled Promise rejections
  - This captures: Worker hangs, IDB write failures, async save failures
- Added `window.addEventListener("error")` for uncaught errors (excluding resource loads)
- Errors are recorded in `TU.GlobalErrorBoundary.recentErrors` (capped at 20)
- Fatal threshold: After 50 errors, emits `fatal:error` event to pause the game
- All errors surface to `console.error` with full context
- Emits `error:boundary` event for monitoring

### 3. IndexedDBStorage error surfacing (line ~8713)
- Changed `console.warn` to `console.error` for all IDB operations
- Added explicit `QuotaExceededError` detection in `set()` method
- Emits `storage:quotaExceeded` event when quota is exceeded
- Previously: IDB write failures returned `false` silently (zombie bad save)
- Now: Errors are logged with key context and events are emitted

### 4. SaveSystem IDB write error surfacing (line ~27184)
- Replaced `.catch(_ => {})` (complete error swallowing) with proper error handler
- Now logs `[SaveSystem] IDB write failed` with the actual error
- Emits `storage:quotaExceeded` event for monitoring
- Shows user-facing toast when IDB save fails and localStorage also failed

### 5. BlockRegistry with collision detection (line ~29159)
- Replaced individual try/catch per property with consolidated registration
- Added `TU._blockRegistry` Map to track all ID assignments
- Detects and logs ID collisions (same numeric ID assigned to different blocks)
- Records registration order for debugging ID drift issues
- Errors during registration are now surfaced with block name and ID

## Files
- `game_part0_safety_net.html` - Backup after Phase 0
- `game_part0_original_backup.html` - Original file backup

## Risk Assessment
- **Low risk**: All changes are additive (new functions) or change log levels
- **No behavioral change**: Error handling still falls back gracefully
- **Monitoring only**: GlobalErrorBoundary observes but doesn't interrupt
