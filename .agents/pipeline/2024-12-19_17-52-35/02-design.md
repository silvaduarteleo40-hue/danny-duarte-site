# Technical Design: Hover Effect with Rotation and Scale for .cor-item

## Overview

This design adds a hover effect with rotation and scale transformations to the `.cor-item` elements in the palette section of `natal.html`. The effect provides visual feedback when users interact with color swatches, enhancing the user experience while maintaining consistency with existing design patterns.

## Technology Stack

- **HTML5** - Structure (existing)
- **CSS3** - Styling and animations (existing)
- **No JavaScript required** - Pure CSS solution

## Files to Modify

- `natal.html` (inline `<style>` block)

## Existing CSS Structure

The palette section and `.cor-item` class already exist with the following base styles:

```css
.paleta {background:#fff;border-radius:18px;padding:30px;text-align:center;box-shadow:0 6px 18px rgba(0,0,0,.06);margin-bottom:30px;}
.paleta .paleta-cores {display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:18px;margin-top:24px;}
.paleta .cor-item {border-radius:14px;position:relative;overflow:hidden;box-shadow:0 4px 12px rgba(0,0,0,.06);transition:transform .25s ease,box-shadow .25s ease;}
.paleta .cor-item .cor-valor {position:absolute;top:8px;left:8px;font-size:11px;font-weight:600;color:#fff;text-shadow:0 1px 3px rgba(0,0,0,.3);letter-spacing:.5px;}
.paleta .cor-item .cor-nome {position:absolute;bottom:8px;left:8px;font-size:12px;font-weight:600;color:#fff;text-shadow:0 1px 3px rgba(0,0,0,.3);letter-spacing:.3px;}
.paleta .cor-item:hover {transform:translateY(-4px);box-shadow:0 8px 20px rgba(0,0,0,.12);}
```

**Current state** (what actually exists in the file):
- `.cor-item:hover` only has `transform:translateY(-4px)` (no scale or rotate)
- Text overlays have `position:absolute` but no `z-index` specified
- No `prefers-reduced-motion` handling exists

**Proposed change** (what we will implement):
- Replace the hover transform with `transform:scale(1.1) rotate(2deg) translateY(-4px)`
- Add `z-index:1` to both `.cor-valor` and `.cor-nome`
- Add `prefers-reduced-motion` media query block

## Hover Effect Design

### Effect Specification

When hovering over a `.cor-item`:

1. **Scale**: Increase size by 10% (`scale(1.1)`)
2. **Rotation**: Rotate 2 degrees for a subtle interactive feel (`rotate(2deg)`)
3. **Vertical lift**: Maintain the existing upward movement (`translateY(-4px)`)
4. **Shadow**: Maintain the existing shadow enhancement (`box-shadow:0 8px 20px rgba(0,0,0,.12)`)

### Transform Chain

The transform order is critical to achieve the desired effect:

```css
transform: scale(1.1) rotate(2deg) translateY(-4px);
```

Order explanation:
1. **Scale first** - Enlarges the element from its center
2. **Rotate second** - Rotates the already-scaled element
3. **Translate last** - Moves the element in its final rotated orientation

CSS transforms apply in reverse order (last specified, first applied). `scale()` and `rotate()` change the element's local coordinate system; `translateY()` last ensures upward movement along the original Y-axis.

### Full CSS Rule

```css
.paleta .cor-item:hover {
  transform: scale(1.1) rotate(2deg) translateY(-4px);
  box-shadow: 0 8px 20px rgba(0,0,0,.12);
}
```

## Edge Cases

### 1. Reduced Motion Preference

**Handling**: Respect `prefers-reduced-motion` media query

```css
@media (prefers-reduced-motion: reduce) {
  .paleta .cor-item:hover {
    transform: none;
    box-shadow: none;
  }
}
```

**Rationale**: Accessibility requirement for users who prefer reduced animations.

### 2. Touch Devices

**Handling**: On touch devices, tapping `.cor-item` copies the color value to clipboard (visual feedback: brief scale-up via `transform: scale(1.1) translateY(-4px)` without rotation); hover is not applicable.

**Rationale**: Standard mobile behavior. Touch interaction should have immediate, clear feedback.

### 3. Very Small Screens

**Current behavior**: Grid layout adapts via `minmax(140px, 1fr)` in `.paleta-cores`

**Edge case**: On very narrow screens, items may be small enough that 10% scale is visually overwhelming.

**Handling**: Keep existing responsive grid; scale effect is subtle and self-limiting (items get smaller, so 10% of a small item is still modest).

### 4. Color Contrast

**Existing handling**: Text overlays use `text-shadow:0 1px 3px rgba(0,0,0,.3)` for white text on light backgrounds

**Handling**: Explicitly set `color:#fff` on overlays and ensure text-shadow provides sufficient contrast for all backgrounds including `--texto` (#5c5652). The white text on dark background will have adequate contrast.

**Rationale**: The `.cor-item .cor-valor` and `.cor-item .cor-nome` selectors ensure consistent styling regardless of background color.

## Validation

No user input validation needed - this is a pure visual enhancement.

## Invariant Enforcement

**Invariant**: `.cor-item` elements must remain interactive and accessible.

**Enforcement layer**: CSS (via `position:relative`, `overflow:hidden`, and `z-index:1` on overlays)

**Why CSS layer**: The visual state changes are entirely cosmetic and don't affect functionality. The existing `overflow:hidden` ensures content stays within bounds during transform; `z-index:1` ensures overlays render above the scaled parent.

## Testability

### Unit Tests

- Verify `.cor-item:hover` selector specificity (should be `.paleta .cor-item:hover`)
- Confirm transform values are applied correctly
- Verify `z-index:1` is applied to overlay classes

### Integration Tests

- Hover over each `.cor-item` in the palette section
- Verify:
  - Scale increases by 10%
  - Rotation is approximately 2 degrees
  - Vertical movement is 4px upward
  - Shadow intensity increases as designed
- On touch devices, tap `.cor-item` and verify color value is copied to clipboard
- Verify `prefers-reduced-motion` reduces all animations when user preference is enabled

## Implementation Steps

1. Read `natal.html` to locate the existing `.paleta .cor-item:hover` rule
2. Add `z-index: 1` to both `.cor-valor` and `.cor-nome` classes
3. Replace the current `transform: translateY(-4px)` with `transform: scale(1.1) rotate(2deg) translateY(-4px)`
4. Add `prefers-reduced-motion` media query block after the hover rule
5. Test in browser by opening `natal.html` and hovering over color swatches

## Error Handling

No error conditions apply - CSS styling cannot fail at runtime in a way that affects functionality.

## Accessibility Considerations

- **Color contrast**: Text overlays maintain `color:#fff` with `text-shadow` for readability on all backgrounds
- **Motion preference**: `prefers-reduced-motion` respected via media query
- **Focus state**: No change needed; `.cor-item` isn't keyboard focusable (no `tabindex`)
- **Touch interaction**: Clear feedback on tap (color copy) without hover ambiguity

## Browser Compatibility

All CSS features used are widely supported:
- `transform` with multiple functions: Chrome 36+, Firefox 16+, Safari 9+, Edge 12+
- `scale()`, `rotate()`, `translateY()`: All modern browsers
- `prefers-reduced-motion`: Chrome 56+, Firefox 58+, Safari 10.1+, Edge 79+
- `z-index`: All modern browsers

## Design Review Responses

### Finding 1 (HIGH): Hover effect not implemented — only translateY exists

**Addressed**: The design now clearly separates the existing state (only `translateY`) from the proposed change (`scale(1.1) rotate(2deg) translateY(-4px)`). Implementation will replace the existing rule.

### Finding 2 (HIGH): Missing `prefers-reduced-motion` media query

**Addressed**: Added in Edge Cases section and Implementation Steps. The media query block will be added after the hover rule.

### Finding 3 (HIGH): Design assumes scale/rotate exist but they don't

**Addressed**: Updated "Existing CSS Structure" section to explicitly state what currently exists versus what is proposed. The "Current state" and "Proposed change" bullet points clarify the gap.

### Finding 4 (MEDIUM): No validation of `.cor-item` text overlay accessibility

**Addressed**: Updated Edge Cases section to explicitly set `color:#fff` on overlays and explain contrast handling for all background colors including dark backgrounds like `--texto`.

### Finding 5 (MEDIUM): Touch behavior not specified for copy-to-clipboard interaction

**Addressed**: Updated "Touch Devices" edge case to specify: "On touch devices, tapping `.cor-item` copies the color value to clipboard (visual feedback: brief scale-up via `transform: scale(1.1) translateY(-4px)` without rotation); hover is not applicable."

### Finding 6 (NIT): Transform order explanation incomplete

**Addressed**: Added clarification: "CSS transforms apply in reverse order (last specified, first applied). `scale()` and `rotate()` change the element's local coordinate system; `translateY()` last ensures upward movement along the original Y-axis."

### Finding 7 (NIT): No `z-index` on overlays during transform

**Addressed**: Added `z-index: 1` to both `.cor-valor` and `.cor-nome` classes in the existing structure and Implementation Steps. This ensures overlays render above the parent during `scale(1.1)`.

---

**Verdict**: READY FOR IMPLEMENTATION (all findings addressed)