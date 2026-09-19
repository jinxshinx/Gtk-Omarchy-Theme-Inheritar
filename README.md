# Gtk-Omarchy-Theme-Inheritar

Gtk-Omarchy-Theme-Inheritar makes ordinary **GTK3**, **GTK3/libhandy**, **GTK4**
and **libadwaita** applications inherit the active **Omarchy** theme palette,
while retaining Omarchy's specialized **Nautilus** palette.

Everything derives from `omarchy-theme-color --file "$COLORS_FILE" --all`, so no
colour is hard-coded and `omarchy theme set <name>` rebuilds every layer.

## What it does

```
Omarchy colors.toml
        │
        ▼
omarchy-theme-color
        │
   ┌────┼──────────────┬───────────────┐
   ▼    ▼              ▼               ▼
 GTK3  libhandy       GTK4          libadwaita
 theme  (priority 800) theme        (priority 800)
 (200)  Disks, Evince,  (200)       GNOME apps
        Seahorse
   └────┴──────────────┴───────────────┘
                    │
                    ▼
            Omarchy palette
                    +
        Nautilus layer (priority 801)
                    │
                    ▼
                Nautilus
```

## Why so many layers

| Layer | Priority | Why it exists |
|---|---|---|
| `~/.local/share/themes/Omarchy-Adwaita/gtk-3.0/gtk.css` | theme (200) | GTK3's normal theme mechanism: Adwaita's own stylesheet with only its palette entries replaced, so geometry, states and accessibility stay Adwaita's |
| `~/.config/gtk-3.0/gtk.css` | user (800) | libhandy apps (GNOME Disks, Evince, Seahorse …) run `HdyStyleManager`, which selects Adwaita itself and layers libhandy's stylesheet — with hard-coded Adwaita colours — at application priority (600). A theme cannot beat that; a user stylesheet can. It contains a **colour-only copy** of the theme's rules (same selectors, same values, no padding/size/border-width/shadow/font declarations), so applications that already use the theme resolve identical values |
| `~/.local/share/themes/Omarchy-Adwaita/gtk-4.0/gtk.css` | theme (200) | Plain GTK4 apps |
| `~/.config/gtk-4.0/gtk.css` | user (800) | libadwaita apps force their theme to `Adwaita-empty`, so a theme can never reach them; GTK4's user stylesheet can. It defines libadwaita's semantic variables (`window/view/headerbar/sidebar/card/dialog/popover/thumbnail`, `accent_*`, `destructive_*`, `success_*`, `warning_*`, `error_*` …) — exactly the same set the Nautilus layer uses |
| `~/.cache/omarchy/gtk/nautilus.css` via `nautilus/omarchy_palette.py` | **801** (`Gtk.STYLE_PROVIDER_PRIORITY_USER + 1`) | Nautilus's own specialized Omarchy palette. Because it outranks the generic 800 layer, Nautilus keeps its own values instead of the generic ones |

Priority matters: in GTK the provider priority is checked before selector
specificity, so a rule at 800 always beats a rule at 600 even if the 600 rule is
more specific — which is exactly what libhandy apps need.

## Nautilus live reload

`nautilus/omarchy_palette.py` runs inside the Nautilus process (nautilus-python),
loads `~/.cache/omarchy/gtk/nautilus.css` into a display-wide provider at
priority 801 and **watches the directory**: hook 70 rewrites that file
atomically on every `omarchy theme set`, and the extension reloads it in place,
so open Nautilus windows restyle immediately without a restart.

## Theme switching

`omarchy theme set <name>` runs the theme-set hooks in order:

```
60-omarchy-gtk-accent.sh   → GSettings accent-color (libadwaita/live)
70-omarchy-nautilus-palette.sh → ~/.cache/omarchy/gtk/nautilus.css
80-omarchy-adwaita-gtk.sh  → generated GTK3 theme, GTK3 user layer,
                             GTK4 theme, GTK4/libadwaita layer
```

Nothing is cached per theme: all five artifacts are regenerated from the staged
`colors.toml`, so light and dark Omarchy themes both work.

## High contrast

If `gtk-theme` is a `HighContrast*` theme, or
`org.gnome.desktop.a11y.interface high-contrast` is true, the generator writes
**neutral, marked** user stylesheets (no palette) and leaves the theme selection
alone, so accessibility palettes are never overridden. Nautilus keeps its own
palette (its hook is independent). Removing the accessibility setting and
re-running `omarchy-adwaita-gtk` restores the Omarchy palette.

## Requirements

- Omarchy 4.x (uses `omarchy-theme-color`, `~/.local/state/omarchy/current/theme/colors.toml`
  and `~/.config/omarchy/hooks/theme-set.d/`)
- GTK3 ≥ 3.24 and GTK4 ≥ 4.10 (assets are read from libgtk's GResource)
- libhandy (for libhandy apps) and libadwaita ≥ 1.4 (for libadwaita apps)
- `python3`, `gresource` (glib2), `gsettings`
- Optional: `nautilus-python` for the Nautilus palette layer

## Installation

```bash
git clone https://github.com/<you>/Gtk-Omarchy-Theme-Inheritar.git
cd Gtk-Omarchy-Theme-Inheritar
./install.sh
```

The installer detects Omarchy and the GTK/libhandy/libadwaita/Nautilus versions,
installs the generator into `~/.local/bin`, the hooks into
`~/.config/omarchy/hooks/theme-set.d/`, the Nautilus extension into
`~/.local/share/nautilus-python/extensions/` (only when no system package
already provides it) and generates the active theme immediately. It is
idempotent and never writes outside your home directory.

## Uninstallation

```bash
./uninstall.sh
```

Removes the generated theme, both user stylesheets, the generator, hook 80 and
(only if this project installed them) hooks 60/70 and the user-local Nautilus
extension, then restores the previous `gtk-theme`. Foreign files, backups,
Lutris configuration and unrelated Nautilus/GTK configuration are untouched.

## Limitations

- **Flatpak applications** get `XDG_CONFIG_HOME=~/.var/app/<id>/config`, so they
  never read `~/.config/gtk-4.0/gtk.css` (or the GTK3 file). Exporting the
  palette into each sandbox is out of scope.
- **Applications with private CSS or hard-coded colours** (Electron/Chromium,
  Firefox, games, GNOME Disks' own widget rules where colours are hard-coded in
  C) keep those colours. Only palette declarations are replaced; no layout is
  touched.
- **libadwaita** only exposes semantic CSS variables; widgets that draw with
  non-semantic colours keep their own values.
- **Nautilus** intentionally keeps its own palette (priority 801) rather than
  the generic one.
- Generated theme files are recoloured copies of the installed GTK stylesheets;
  run `omarchy-adwaita-gtk` after a GTK upgrade to pick up a new base sheet.

## Documentation

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — layers, priorities, files,
  generation flow
- [`docs/NAUTILUS.md`](docs/NAUTILUS.md) — the Nautilus layer, live reload and
  its relationship to `paint-omarchy-nautilus`
- [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) — symptom → cause → fix

## Licence

MIT — see [LICENSE](LICENSE). The Nautilus integration files are derived from
[paint-omarchy-nautilus](https://github.com/JJDizz1L/paint-omarchy-nautilus)
(MIT, © 2024 JJDizz1L); their notice is retained in `LICENSE` and in the file
headers.
