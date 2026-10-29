# Design Review: Hover Effect with Rotation and Scale for .cor-item

## Findings

1. **HIGH — Hover effect implementation unclear**  
   **Location**: `02-design.md`, Implementation Steps, Transform Chain section  
   **Problem**: The design states `transform: scale(1.1) rotate(2deg) translateY(-4px)` will be applied, but the explanation about "reverse order" is technically correct yet misleading. It says "CSS transforms apply in reverse order (last specified, first applied)" — this is correct, but the phrasing "original Y-axis" is inaccurate. `translateY` operates on the *already transformed* coordinate system, not a hypothetical "original" axis.  
   **Fix**: Replace the current explanation with:  
   > "CSS transforms are applied right-to-left. So `scale(1.1) rotate(2deg) translateY(-4px)` means: (1) `translateY` moves the element first, (2) `rotate` turns the translated result around its center, (3) `scale` enlarges the rotated result. The final visual is: element lifts 4px, rotates 2°, then grows by 10%."

2. **HIGH — No JavaScript for copy-to-clipboard touch interaction**  
   **Location**: `02-design.md`, "Touch Devices" Edge Case, Implementation Steps  
   **Problem**: The design assumes "On touch devices, tapping `.cor-item` copies the color value to clipboard" with visual feedback, but there is *zero* JavaScript in `natal.html` that handles `.cor-item` taps. The only JS in the file controls music playback.  
   **Fix**: Either (a) remove the clipboard behavior from the design if out of scope, or (b) specify minimal JS in the design:  
   > "Add a `click` handler for `.cor-item` elements that calls `navigator.clipboard.writeText(colorValue)` and briefly applies `.active` class (e.g., `transform: scale(1.1) translateY(-4px); transition: transform 0.15s ease;`) for touch feedback."

3. **HIGH — Contrast risk for dark background swatch (`--texto`)**  
   **Location**: `natal.html`, `.paleta .cor-item .cor-valor`, `.paleta .cor-item .cor-nome`  
   **Problem**: The `.cor-item` with background `var(--texto)` (`#5c5652`) uses white text (`color:#fff`) with `text-shadow:0 1px 3px rgba(0,0,0,.3)`. The `text-shadow` darkens the text edges, *worsening* contrast. Contrast ratio for white text on `#5c5652` is ~3.1:1, which fails WCAG AA (minimum 4.5:1 for normal text).  
   **Fix**: Specify a concrete solution in the design:  
   > "For `.cor-item` with dark background (`#5c5652`), override text styling: `.paleta .cor-item.dark .cor-valor, .paleta .cor-item.dark .cor-nome { color:#fff; text-shadow: 0 1px 2px rgba(0,0,0,.6); }` OR use `-webkit-text-stroke: 0.5px #2a2522;` to increase legibility while keeping `color:#fff`."

4. **MEDIUM — `prefers-reduced-motion` media query missing implementation scope**  
   **Location**: `02-design.md`, Edge Cases, "Reduced Motion Preference"  
   **Problem**: The design proposes adding `prefers-reduced-motion: reduce` but does not clarify whether *all* transitions (e.g., `transition: transform .25s ease, box-shadow .25s ease` on `.cor-item`) should be disabled or only the hover effects.  
   **Fix**: Add to the design:  
   > "When `prefers-reduced-motion: reduce`, disable all animations:  
   > `.paleta .cor-item { transition: none; }`  
   > `.paleta .cor-item:hover { transform: none; box-shadow: none; }`"

5. **MEDIUM — Touch feedback visual state not specified**  
   **Location**: `02-design.md`, "Touch Devices" Edge Case  
   **Problem**: The design says touch tap shows "brief scale-up via `transform: scale(1.1) translateY(-4px)` without rotation", but CSS only has a hover state. There is no `:active` or `.active` class defined for touch feedback, so hover and tap effects would be identical in code, causing confusion.  
   **Fix**: Add to the design:  
   > "Define separate visual states:  
   > - Hover: `.paleta .cor-item:hover { transform: scale(1.1) rotate(2deg) translateY(-4px); }`  
   > - Active (touch): `.paleta .cor-item:active { transform: scale(1.05) translateY(-2px); }`  
   > This ensures touch feedback is subtle and distinct."

6. **NIT — `z-index: 1` alone may not prevent overlay clipping**  
   **Location**: `natal.html`, `.paleta .cor-item` and overlays  
   **Problem**: `.cor-item` has `overflow:hidden`. When scaled by 1.1, content may be clipped by the border-radius. `z-index: 1` on overlays does not fix clipping.  
   **Fix**: Add to the design:  
   > "Two options to prevent clipping:  
   > (a) Remove `overflow:hidden` from `.cor-item` and keep `border-radius:14px` only on the parent, or  
   > (b) Add padding to `.cor-item` to accommodate the 10% scale growth (e.g., `padding:10px` instead of relying on absolute-positioned overlays)."

---

## Verified Assumptions

- ✅ `.cor-item:hover` currently has only `transform:translateY(-4px)` — verified in source.
- ✅ No `z-index` on `.cor-valor` or `.cor-nome` — verified.
- ✅ No `prefers-reduced-motion` media query — verified.
- ✅ Grid uses `minmax(140px, 1fr)` — verified.
- ✅ Background for `--texto` is `#5c5652` — verified from `:root` and DOM.

## Unverified/Wrong Assumptions

- ❌ **Touch clipboard behavior exists or is planned** — no JavaScript handles `.cor-item` clicks. The file contains only music playback JS.
- ❌ **White text with `text-shadow` provides sufficient contrast for dark backgrounds** — `#5c5652` + white text + `text-shadow:0 1px 3px rgba(0,0,0,.3)` = ~3.1:1 ratio (FAILS WCAG AA 4.5:1).
- ❌ **Scale/rotate transform exists or is planned with touch fallback** — design references non-existent behavior.
- ❌ **No `prefers-reduced-motion` scope specified** — unclear if it applies only to hover or all transitions.

---

**Verdict**: CHANGES_REQUESTED (4 HIGH, 2 MEDIUM, 1 NIT findings)