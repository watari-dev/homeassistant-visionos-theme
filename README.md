# visionOS & iOS 26 Liquid Glass Theme

Theme inspired by visionOS for Home Assistant with automatic dark mode support.

The performance defaults keep the wallpapers, colors and rounded surfaces, but
use tinted cards instead of a live blur behind every card. The header, sidebar
and dialogs retain a small blur. The screenshots below show the original
full-blur appearance; current cards have a stronger tint.

See the [full performance audit](docs/PERFORMANCE_AUDIT.md) for findings,
measurements, tradeoffs and limitations.

### visionOS
<img width="500" alt="vision-light" src="https://github.com/user-attachments/assets/f054c59e-7198-4476-9a2e-4e0caec49df8" /><img width="500" alt="vision-dark" src="https://github.com/user-attachments/assets/61179b34-d25b-4902-9883-91156f5dc659" />

### Liquid Glass
<img width="500" alt="ios-light" src="https://github.com/user-attachments/assets/c60d760b-4531-41c2-b8b5-47404e8743d7" /><img width="500" alt="ios-dark" src="https://github.com/user-attachments/assets/273f0e86-180e-42b3-abe0-bab25c359584" />

## Installation

1. You can install the theme with [HACS](https://hacs.xyz/docs/setup/download):

[![Open your Home Assistant instance and open a repository inside the Home Assistant Community Store.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=Nezz&repository=homeassistant-visionos-theme&category=theme)

> [!NOTE]  
> Install the [`uix`](https://github.com/Lint-Free-Technology/uix) integration via HACS
> for the optional sidebar/drawer styling. It is a replacement for card-mod;
> follow its [migration/setup guide](https://uix.lf.technology/quick-start/#add-ui-extension-service)
> and do not load both injectors. The native card performance defaults also work
> without either injector. Existing card-mod installations can keep card-mod;
> drawer support varies with the Home Assistant/injector version.

2. You should see the "Liquid Glass" and "visionos" themes appear in your list of themes.

If it's missing, try reloading your themes or adding the following code to your `configuration.yaml` file (reboot required):

```yaml
frontend:
  themes: !include_dir_merge_named themes
```

3. (Optional) You can set this as the default theme with the following automation:
```
alias: Frontend - Change theme
trigger:
  - platform: homeassistant
    event: start
action:
  - service: frontend.set_theme
    data:
      name: visionos
```

## Performance and customization

- Native cards no longer create a backdrop filter by default. This also covers
  cards before the styling injector loads and cards that it does not support.
- No extra `ha-card::before` blur layer or duplicate shadow is added.
- Liquid Glass uses native slider shapes instead of searching inside every card
  for five possible slider types. Sliders keep their normal actions.
- Header/sidebar/dialog blur is separate from card blur. To disable that too,
  change `glass-chrome-backdrop-filter` to `none` in both theme files.
- Keep one injector and one version. If using card-mod, follow its
  [frontend-module installation instructions](https://github.com/thomasloven/lovelace-card-mod#performance-improvements).
  The `extra_module_url` and HACS resource URL must match exactly to prevent
  duplicate loading. Changing `extra_module_url` requires a Home Assistant restart;
  changing these theme files only requires `frontend.reload_themes` and a browser refresh.

For an optional full-blur look, set `ha-card-backdrop-filter: blur(8px)` and reduce
`ha-card-background` opacity in each mode. This restores the per-card rendering
cost and native backdrop stacking behavior, so it is not recommended on devices
that stall with the original theme.

Wallpapers are hosted remotely. To use your own local images, place them under
`/config/www/` and set the `background-image` value in each mode, for example:

```yaml
background-image: "center / cover no-repeat fixed url('/local/wallpapers/day.webp')"
```

Keep a copy of your custom values: HACS upgrades replace installed theme files.
You can return to Home Assistant's built-in theme from your profile at any time.

## Remarks

Sample dashboard configuration from the themes is available [here](https://github.com/Nezz/homeassistant-visionos-theme/blob/sample/sample.yaml)

Based on [Bas Nijholt's iOS Themes](https://github.com/basnijholt/lovelace-ios-themes)

Dropdown fixes from [Wessam Lauf's Frosted Glass Theme](https://github.com/wessamlauf/homeassistant-frosted-glass-themes)
