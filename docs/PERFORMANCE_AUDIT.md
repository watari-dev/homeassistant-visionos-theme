# Theme performance audit

Audit date: 2026-09-21. Upstream: `Nezz/homeassistant-visionos-theme`,
release **3.0.7**, commit `fe496373ba0a74eefd41475ec4c6801ee988c678`.
Both installed YAML files matched that commit byte for byte.

## Scope and conclusion

Reviewed every tracked file: both theme definitions, README, HACS manifest,
license and validation workflow. Also checked the actual Home Assistant frontend
20260826.7 (Core 2026.9.3), installed card-mod 4.2.1, the live dashboard, and
upstream reports [#26](https://github.com/Nezz/homeassistant-visionos-theme/issues/26)
and [#66](https://github.com/Nezz/homeassistant-visionos-theme/issues/66).

The theme is configuration and CSS, not a server integration. It has no polling
loop, entity subscription, automation, database query, bundled JavaScript or
server process of its own. Its main scaling costs are browser backdrop rendering
and work requested from card-mod. A responsive server can coexist with a slow UI.

**The reported unusable state was not reproduced in the short desktop checks.**
Consequently this audit does not claim a proven cause for every reported freeze,
a specific Home Assistant 2026.8 regression, a memory leak, or an FPS improvement.
It identifies and removes avoidable costs in the theme, and verifies the reduction
in the real dashboard. Mobile/iOS performance still needs device-specific validation.

## Findings

### 1. Global blur scales with every card — high impact risk, confirmed live

Both themes set `ha-card-backdrop-filter` globally: 20 px for visionos and 8 px for
Liquid Glass. Home Assistant's native `ha-card` uses that variable directly,
including cards outside Lovelace, nested custom cards, and cards whose card-mod
styles have not loaded. Each such surface must filter the pixels behind it.
Large dashboards, high pixel density, moving content and nested surfaces amplify
that work. File size is not a useful proxy for this rendering cost.

The inspected desktop dashboard had **35 native card blur surfaces**, plus other
filtered surfaces, with the original Liquid Glass theme. Some card-mod card
instances had empty resolved styles, so the native variable remained active even
when the theme's intended pseudo-element workaround was not applied. This makes a
fix relying exclusively on an injected CSS override unreliable.

**Fix:** set the native card filter to `none`. Use a more opaque 78% card tint to
keep labels readable over the wallpaper. Introduce an independent
`glass-chrome-backdrop-filter: blur(8px)` for the header, sidebar and open dialog.
This keeps the small, bounded set of chrome surfaces independent of card count.
The card change works before card-mod loads and without card-mod.

**Tradeoff:** card surfaces are tinted rather than live-frosted. This is a deliberate
performance default, not a claim of pixel-identical rendering.

### 2. Extra negative-z-index surfaces and duplicate shadows — confirmed source

The original injected CSS disables the native card filter, then creates an
absolutely positioned `ha-card::before` with a negative z-index, another filter
and the same shadow that Home Assistant already draws on the card host. It needs
exceptions for headings, text-only Markdown, Mushroom titles and chips.

When applied, this adds another surface and repeats the shadow. Negative stacking
also makes the result depend on ancestor stacking contexts. In the observed
session this workaround was not reliably active; it must not be counted as an
additional live blur on every observed native card.

**Fix:** remove the pseudo-element surface. Native cards draw one tinted surface
and one shadow. Consolidate transparent-card exceptions, including disabling the
remaining native shadow on text/title-only surfaces.

### 3. Global shadow-tree searches for five slider types — confirmed source

Liquid Glass applies `hui-card-features $` to every styled card, descends through
`hui-card-feature $`, and then tries five feature branches just to change slider
corner radii. Most cards do not contain those features, and most feature elements
match only one branch.

In [card-mod 4.2.1](https://github.com/thomasloven/lovelace-card-mod/blob/v4.2.1/src/card-mod.ts),
child-style paths cause mutation observation and selector retries. The
[`selectTree` helper](https://github.com/thomasloven/lovelace-card-mod/blob/v4.2.1/src/helpers/selecttree.ts)
also sets a 10-second timeout for each call. A missing match can be attempted seven
times by `_style_child` (initial attempt and six retries). This is avoidable setup
and navigation work, not evidence of an infinite loop or a confirmed memory leak.

The live card-mod styles were empty on sampled cards; these searches were therefore
not established as the active cause of the observed session's responsiveness.
They remain a theme-level cost when the styles do apply, including with UIX.

**Fix:** use flat `card-mod-card` CSS. Let Home Assistant and custom cards keep
their native sliders. Slider actions and entity configuration are unchanged.

### 4. Sidebar filtering and private shadow access — confirmed source

The original theme requests blur on the sidebar, legacy drawer, and the dialog
inside a Web Awesome drawer. Depending on layout and injector support, surfaces
can overlap. The `wa-drawer$` selector creates another shadow-tree traversal.

**Fix:** keep blur on the sidebar only. Drawer background rules use flat CSS and
the public `wa-drawer::part(dialog)` selector. The old `.mdc-drawer` selector stays
for older frontends. This removes theme-owned drawer child searches.

### 5. Three malformed CSS values — low severity, fixed

- `rgb-card-background-color` included `rgb(...)`, although consumers use it as
  channels inside `rgba(var(--rgb-card-background-color), alpha)`; use `10, 10, 10`.
- `md-list-container-color: none` is not a color; use `transparent`.
- `ha-slider-background` embedded `!important` inside a theme value; use `none`.
  Priority belongs to a CSS declaration, not a value passed through the theme API.

These are correctness fixes. No measurable performance gain is attributed to them.

## Other areas reviewed

| Area | Result |
| --- | --- |
| Wallpapers | HTTPS remote image dependencies remain. The checked visionos day JPEG is 872,419 bytes; Liquid Glass dark WebP is 119,666 bytes, each served with a five-minute cache policy. They can add cold-load/network cost, but no causal evidence tied them to ongoing freezes. Local wallpaper instructions are provided. |
| Fixed backgrounds | Retained to preserve wallpaper behavior, including the project's original iOS scroll fix. No unmeasured claim that changing background attachment would help. |
| Animations | No theme-owned animation loop, timer or `will-change` layer promotion. Native HA component transitions remain. |
| Templates | No Jinja templates or template-driven server subscriptions in either theme. |
| Dependencies | The theme needs no build or runtime package installation. UIX/card-mod are external injectors. Do not load both together; the theme fix does not silently replace either. |
| HACS metadata | Valid theme layout and two-mode definitions. The existing minimum Core version is not proof of compatibility with every optional modern CSS feature. Older drawers retain a fallback selector. |
| Workflow | Existing HACS validation checks packaging, not responsiveness. No additional hosted benchmark workflow is added. |
| Security/privacy | No executable code or secrets bundled. Remote wallpapers disclose normal image-request metadata to their host. License and upstream attribution remain. |
| Maintainability | Flat CSS replaces repeated exceptions and nested selector trees; no extra runtime or compatibility framework. |

## Live validation

Client: desktop Chrome, macOS, viewport 1486 × 963 CSS pixels, device pixel ratio 2.
The same Sensors view, same live data and same browser were used. Theme selection
was performed through the normal Home Assistant profile UI. Production device
controls were not activated.

| Observation | Original Liquid Glass | Optimized Liquid Glass |
| --- | ---: | ---: |
| Native cards present | 35 | 35 |
| Native cards with an active backdrop filter | 35 | 0 |
| Elements with an active blur filter | 42 | 2 |
| Theme-requested card child selector branches | Present in configuration | None |
| Theme-created card pseudo-element surface | Requested, not reliably applied in this session | Not requested |

A short requestAnimationFrame-driven scroll check over the same 958 px scroll
range yielded p95 frame intervals of 9.0 ms before and 9.3 ms after. Both were
already near the display cadence. One optimized sample contained a 66.7 ms outlier;
the original sample's maximum was 16.6 ms. These short, live-data samples **do not
show an FPS improvement** and cannot establish a regression either. A separate
variable-only blur-off check also did not improve frame cadence. The confirmed
result is removal of **35 card filters / 40 total filtered elements** from this
view, not a throughput claim.

The 35-card case also showed that the filter reduction survives a full page load,
without depending on card-mod's global card styling. Wallpapers, graphs, entity
labels and layout were checked visually. See the deployment notes for additional
mode, navigation and reload checks performed on the installed fork.

## Reproducing a fair performance comparison

1. Record the exact HA, browser, card-mod/UIX and theme versions. Keep the same
   viewport, zoom, dashboard, live-data load and power settings.
2. Save the original theme selection, choose one theme, reload, and allow custom
   cards/history graphs to settle before measuring.
3. Count computed `backdrop-filter` values across shadow roots; distinguish native
   cards, pseudo-elements and header/sidebar surfaces. A YAML change alone is not
   live proof.
4. Alternate original/fixed/original/fixed scroll and navigation captures. Use a
   browser performance trace on the affected device for long tasks, raster/GPU
   work and frame stalls; do not substitute a Lighthouse load score for runtime
   responsiveness.
5. Exercise menus, text-only cards, native sliders, light/dark mode, narrow layouts
   and non-dashboard panels. Do not actuate household devices just to test CSS.
6. If freezing persists with the fixed theme or the stock HA theme, inspect custom
   card errors, duplicate resources, history-card payloads and frontend regressions
   separately. They are outside this theme repository.
