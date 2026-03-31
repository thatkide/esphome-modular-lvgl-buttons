# CLAUDE.md — esphome-modular-lvgl-buttons

This is an **ESPHome YAML library** for building LVGL touchscreen UIs on ESP32 devices connected to Home Assistant. Not Python, not JavaScript — pure ESPHome YAML with inline C++ lambdas.

Read ARCHITECTURE.md for the full architecture reference.

## Build / Test

```bash
# Validate all example configs
cd /path/to/parent/directory  # directory containing esphome-modular-lvgl-buttons/
bash esphome-modular-lvgl-buttons/testing/test_configs.sh

# Validate a single config
esphome config example_code/SDL-lvgl-display_modular_480px.yaml
```

## Essential Patterns

- **Always use named colors, never hex values.** Use library colors from `common/color.yaml` (e.g., `ep_orange`, `misty_blue`, `very_dark_gray`, `gray50`-`gray900`) or ESPHome built-in CSS color names (e.g., `coral`, `dodgerblue`, `teal`, `tomato`). Full list in `common/color.yaml`. If you need a new color, define it as a substitution (e.g., `my_teal: "0x1ABC9C"`) and reference as `$my_teal` — never inline hex.
- Every button needs a **unique `uid`** — generates LVGL widget IDs: `button_${uid}`, `icon_${uid}`, `label_${uid}`, `widget_sensor_${uid}`
- Buttons extend pages: `!extend ${page_id | default("main_page")}` — never create standalone pages
- Variables: `${var | default(value)}` syntax
- Colors via substitution vars: `$button_on_color`, `$button_off_color`, `$icon_on_color`, `$icon_off_color`, `$label_on_color`, `$label_off_color`
- Icons from `common/mdi_glyph_substitutions.yaml`: `$mdi_lightbulb`, `$mdi_home`, etc.
- Fonts: `$icon_font` for icons, `$text_font` for labels
- Grid positioning: `grid_cell_row_pos`, `grid_cell_column_pos` (0-based), optional `grid_cell_row_span`/`grid_cell_column_span`
- HA state tracking: `binary_sensor` (on/off) or `text_sensor` (multi-state) from `platform: homeassistant`

## New Button Checklist

1. Create file in `buttons/` with YAML comment header listing `vars:`
2. `lvgl: pages:` using `!extend` → `container` → `button` → labels
3. Add `binary_sensor` or `text_sensor` for HA state tracking
4. Update colors in state handler with `lvgl.widget.update`
5. Test with SDL: add to `example_code/SDL-lvgl-display_modular_480px.yaml`, run `esphome config`

## Key Files

- `buttons/switch_button.yaml` — canonical simple button pattern
- `buttons/cover_button.yaml` — complex multi-state example
- `common/theme_style.yaml` — LVGL theme defaults and Dark/Light switcher
- `common/mdi_glyph_substitutions.yaml` — ~6000 icon substitution mappings
- `common/color.yaml` — ~50 named colors
- `common/fonts.yaml` — Nunito font family (sizes 12-72)
- `hardware/` — one file per supported display device (27+)

## LVGL Gotchas

Read **LVGL_REFERENCE.md** for critical LVGL v8 behavior — especially image tiling, layout override rules, and the obj wrapper pattern for images.

## Page Grid Layout Pattern

When using a grid layout on a page, use the shorthand `layout: NxM` (rows x columns) and place children with `grid_cell_row_pos` / `grid_cell_column_pos`. **Every child must have both `grid_cell_x_align: stretch` and `grid_cell_y_align: stretch`** so LVGL sizes them to fill their cells. Without these, widgets collapse to content size and the grid looks broken.

```yaml
# Example: 3x3 page grid
lvgl:
  pages:
    - id: main_page
      layout: 3x3
      styles: page_style
      widgets:
        - obj:
            grid_cell_row_pos: 0
            grid_cell_column_pos: 0
            grid_cell_x_align: stretch
            grid_cell_y_align: stretch
            # ... widget content
        - obj:
            grid_cell_row_pos: 0
            grid_cell_column_pos: 1
            grid_cell_row_span: 2      # span multiple cells
            grid_cell_column_span: 2
            grid_cell_x_align: stretch
            grid_cell_y_align: stretch
            # ... widget content
```

## ESPHome Jinja/Substitutions

ESPHome uses Jinja2 but with **different delimiters** than standard Jinja. Getting this wrong produces broken YAML.

### Expression Syntax
- ESPHome uses `${...}` instead of `{{ ... }}` for expressions
- **`{% if %}` / `{% else %}` blocks are NOT supported** — use inline ternary instead

### Inline Ternary (the correct way)
```yaml
bg_color: ${"gray200" if style | default("black") == "white" else "gray900"}
text_color: ${"gray900" if style | default("black") == "white" else "gray400"}
width: ${native_width * 2 if high_dpi else native_width}
hidden: ${not debug_label.enabled}
```

### Variable Access
- Simple: `$var` or `${var}`
- Dict member: `${device.name}` or `${device["name"]}`
- List index: `${unused_pins[2]}`

### !include vars with defaults
Variables passed via `vars:` in `!include` are accessed directly by name. Use `| default()` for fallbacks:
```yaml
# In the including file:
button: !include
  file: widgets/flip_clock.yaml
  vars:
    uid: button_2
    style: white

# In the included file:
bg_color: ${"gray200" if style | default("black") == "white" else "gray900"}
```

### Available Filters & Functions
- Jinja filters: `| round`, `| int`, `| default()`, `| upper`, `| lower`
- Python `math` module: `${math.sqrt(x*x+y*y)}`
- Built-ins: `ord()`, `chr()`, `len()`

### Disable Substitution
Use `!literal` tag to prevent variable expansion:
```yaml
text: !literal "This is a ${value}"  # outputs literally: This is a ${value}
```

Reference: https://esphome.io/components/substitutions/

## Known Issues

- `shadow_width: 0` must be explicitly set in theme despite LVGL default
