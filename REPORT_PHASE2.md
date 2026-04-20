# Phase 2 Report: PostFX Pipeline Flattening

## Completion: 100%

## What was done

### Problem: 4-Layer Onion Model
The `Renderer.applyPostFX` method was wrapped 4 times by successive patches:

| Layer | Line | Flag | Purpose |
|-------|------|------|---------|
| Base | 20836 | (native class) | Bloom, fog, vignette, color grading (comprehensive) |
| Layer 1 | 26510 | (replaces entirely) | Simpler vignette/grain/warm/cool |
| Layer 2 | 28512 | `__weatherPostTintInstalled` | Weather color tint (rain/thunder/bloodmoon) |
| Layer 3 | 28987 | `__weatherPostTintOptimized` | "Optimized" weather tint that SUPPRESSES Layer 2 to avoid double-apply |
| Layer 4 | 29680 | `__underwaterFogInstalled` | Underwater fog + deep tint overlay |

**Call chain**: Render call -> Layer 4 -> Layer 3 -> Layer 2 -> Layer 1

Layer 3 had to temporarily zero-out fx.postA and fx.lightning before calling Layer 2, then restore them after - this is the classic "patches suppressing each other" anti-pattern.

### Solution: Unified Linear Pipeline

Replaced Layer 1 with a single comprehensive function containing 3 explicit stages:

1. **STAGE 1: Base PostFX** - Vignette, grain texture, warm/cool/fog color overlays (from Layer 1)
2. **STAGE 2: Weather Tint** - Rain/thunder/bloodmoon color overlay + lightning flash (from Layer 3's optimized logic)
3. **STAGE 3: Underwater Fog** - Deep underground tint + water submersion overlay (from Layer 4)

### Neutralization of old layers

Layers 2, 3, and 4 are neutralized by adding `&& false` to their guard conditions:
- `!Renderer.prototype.__weatherPostTintInstalled && false /* [Phase2] neutralized */`
- `!Renderer.prototype.__weatherPostTintOptimized && false /* [Phase2] neutralized */`
- `!Renderer.prototype.__underwaterFogInstalled && false /* [Phase2] neutralized */`

This ensures the old wrapping code never executes while remaining in the file for reference during validation.

## Performance Impact
- **Before**: 4 nested function calls per frame, each with its own prototype lookup, `prev.call(this, ...)`, and context save/restore
- **After**: 1 function call with 3 linear stages, single prototype lookup
- **Eliminated**: Layer 3's hack of temporarily zeroing weather params to suppress Layer 2
- **Eliminated**: 3 redundant `ctx.save()/ctx.restore()` pairs (one per wrapper layer)

## Files
- `game_part2_pipeline.html` - Backup after Phase 2

## Risk Assessment
- **Medium risk**: The unified function must perfectly replicate the behavior of the 4-layer chain
- **Mitigated**: Layer 3's logic was the one that actually ran (it suppressed Layer 2), so only Layer 3's weather tint behavior is preserved
- **Validation needed**: Weather effects (rain tint, thunder lightning, bloodmoon), underwater fog, and vignette should all render correctly
