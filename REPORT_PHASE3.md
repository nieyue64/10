# Phase 3 Report: Canvas getContext Isolation + Weather State Consolidation

## Completion: 100%

## What was done

### 1. Canvas getContext Global Hijack Replaced (line ~5490)

**Problem**: `HTMLCanvasElement.prototype.getContext` was globally overridden to inject caching for `globalAlpha` and `fillStyle` on ALL 2D canvas contexts in the page. This:
- Modifies a native Web API prototype (blocks future browser optimizations)
- Applies to non-game canvases (minimap, offscreen buffers, UI icons) that don't benefit
- Has a subtle state tracking bug: the save/restore stack only tracks alpha/fill, not compositeOperation, strokeStyle, etc.

**Fix**: 
- Replaced global prototype hijack with `window.TU.CanvasOptimizer.wrap(ctx)` utility
- The utility applies the same caching optimization but on individual context instances
- `CanvasOptimizer.wrap()` is called only on the game canvas context in the Renderer constructor
- `HTMLCanvasElement.prototype.getContext` is no longer modified
- Property descriptors are marked `configurable: true` for debuggability

### 2. Renderer Constructor Updated (line ~18557)
- Added `window.TU.CanvasOptimizer.wrap(this.ctx)` after context acquisition
- The game's main rendering context gets the performance optimization
- Other canvases (minimap, chunk cache, UI) use native unmodified contexts

### 3. Weather State Consolidation (line ~26698)

**Problem**: Weather state was split between two objects:
- `game.weather` - type, intensity, targetIntensity, lightning
- `window.AppServices.get('weatherFx')` - postA, postR/G/B, lightning, gloom, shadowColor

These were created independently and updated separately, making it hard to reason about weather state.

**Fix**: 
- `game.weather.fx` is now set to reference the same object as `AppServices.get('weatherFx')`
- If `weatherFx` service already exists, `game.weather.fx` points to it
- If not, `game.weather.fx` creates it and registers it as the service
- This establishes a single source of truth: `game.weather` contains everything
- All existing code using `AppServices.get('weatherFx')` continues to work unchanged

## Files
- `game_part3_canvas_weather.html` - Backup after Phase 3

## Risk Assessment
- **Low risk**: Canvas optimizer is functionally identical, just scoped to game canvas
- **Low risk**: Weather consolidation is additive (adds .fx reference, doesn't change existing paths)
- **Note**: Other canvases that previously got auto-optimized now run with native performance (slightly different, but more predictable)
