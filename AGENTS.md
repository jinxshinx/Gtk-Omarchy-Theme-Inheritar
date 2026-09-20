# AGENTS.md

Instructions for AI coding agents working on this repository. Read this before
changing anything. `README.md` is the user-facing document; `docs/` explains the
architecture in depth. This file records what an agent needs in order to make
correct changes without undoing deliberate decisions.

## What this project is

An **Omarchy-specific integration**, not a general-purpose GTK theme. It makes
GTK3, GTK3/libhandy, GTK4 and libadwaita applications inherit the active Omarchy
palette derived from `colors.toml`, while Nautilus deliberately keeps its own
specialized Omarchy palette at a higher provider priority.

Everything is generated; nothing about a particular theme (Nord Zelda,
Catppuccin, an accent, a background) may be hard-coded.

## Repository map

| Path | Role |
|---|---|
| `bin/omarchy-adwaita-gtk` | The generator (~1300 lines, stdlib + optional `gi`). The only place theming logic lives. |
| `hooks/80-omarchy-adwaita-gtk.sh` | Tiny theme-set hook: calls the generator with `--quiet`. |
| `hooks/60-omarchy-gtk-accent.sh`, `hooks/70-omarchy-nautilus-palette.sh` | Nautilus/accent integration, derived from `paint-omarchy-nautilus` (MIT, © 2024 JJDizz1L). Preserved as-is. |
| `nautilus/omarchy_palette.py` | Nautilus-python extension: display-wide CSS provider at priority 801 + directory watcher for live reload. |
| `install.sh`, `uninstall.sh` | Idempotent installer / scoped uninstaller (marker + state-file driven). |
| `docs/ARCHITECTURE.md`, `docs/NAUTILUS.md`, `docs/TROUBLESHOOTING.md` | Layer-by-layer explanation, Nautilus specifics, symptom→cause→fix table. |
| `docs/screenshots/` | Illustrative screenshots; not generated output. |

## What an installation produces

| Artifact | Provider priority | Purpose |
|---|---|---|
| `~/.local/share/themes/Omarchy-Adwaita/gtk-3.0/gtk.css` | theme (200) | GTK3: recoloured copy of the installed Adwaita sheet |
| `~/.config/gtk-3.0/gtk.css` | user (800) | Colour-only copy of the theme rules, so **libhandy** apps (Disks, Evince, Seahorse) get the palette |
| `~/.local/share/themes/Omarchy-Adwaita/gtk-4.0/gtk.css` | theme (200) | Plain GTK4 apps |
| `~/.config/gtk-4.0/gtk.css` | user (800) | **libadwaita** semantic variables (it forces `Adwaita-empty`, so a theme can never reach it) |
| `~/.cache/omarchy/gtk/nautilus.css` | **801** (`PRIORITY_USER + 1`) | Nautilus's own palette, written by hook 70 and hot-reloaded by the extension |
| `~/.local/state/gtk-omarchy-theme-inheritar/install.json` | — | What `install.sh` installed (used by `uninstall.sh`) |

Hook order on `omarchy theme set`: **60 accent → 70 Nautilus palette → 80 GTK
inheritance**. Do not reorder or merge them; 80 must run last so the layers are
regenerated from the freshly staged `colors.toml`.

## Pipeline

```
~/.local/state/omarchy/current/theme/colors.toml
  → omarchy-theme-color --file <file> --all          (palette: never parsed by hand)
  → build_mapping()                                   (semantic roles)
  → render_css / render_gtk4_css                       (recoloured Adwaita copies)
  → render_gtk3_user_css / render_libadwaita_css       (user-priority layers)
  → atomic writes + hook installation + gtk-theme selection
```

Other hooks remain authoritative for their own artifacts: 60 owns
`org.gnome.desktop.interface accent-color`, 70 owns `nautilus.css`.

## Design decisions and why (do not undo these)

1. **Recolour Adwaita; do not author widget CSS.** GTK 3.24/4.x inline literal
   colours, so overriding `@define-color` alone does not restyle anything
   (measured: `theme_bg_color` changed while the window stayed `#353535`).
   The generator substitutes palette entries inside a copy of the installed
   stylesheet, which preserves geometry, states and accessibility.
2. **Rewrite asset URLs to `resource://`.** Copied sheets otherwise point at a
   non-existent `assets/` directory and GTK paints its red broken-image
   placeholder over indicators (`check`, `radio`, sliders). GTK3 →
   `resource:///org/gtk/libgtk/theme/Adwaita/assets/`, GTK4 → `…/Default/assets/`.
3. **The GTK3 user layer exists because of libhandy.** `HdyStyleManager` chooses
   Adwaita itself and layers libhandy's stylesheet (hard-coded Adwaita colours)
   at application priority 600; a theme (200) cannot win, a user stylesheet
   (800) can. It must stay **generic** — no per-application selectors.
4. **The GTK3 user layer is colour-only.** It lifts `background`,
   `background-color`, `background-image`, `color`, `border-*-color`,
   `outline-color`, `caret-color`, `-gtk-secondary-caret-color` and nothing
   else, so application geometry
   stays untouched and apps already using the theme render identically.
   `background` is required — Adwaita styles headerbars with that shorthand, and
   dropping it reverted libhandy headerbars to Adwaita colours.
5. **libadwaita needs GTK4 user CSS.** libadwaita forces `gtk-theme-name =
   Adwaita-empty`; no theme can reach it. `~/.config/gtk-4.0/gtk.css` defines
   its semantic variables (`window/view/headerbar/sidebar/card/dialog/popover/
   thumbnail_*`, `accent_*`, `destructive_*`, `success_*`, `warning_*`,
   `error_*`), using the same variable set the Nautilus layer uses.
6. **Nautilus at 801 is intentional.** Its extension registers at
   `Gtk.STYLE_PROVIDER_PRIORITY_USER + 1`, so Nautilus keeps its own palette
   instead of the generic one. Never "unify" the layers, and never move the
   generic layer above 800.
7. **HighContrast wins.** If `gtk-theme` is `HighContrast*` or
   `org.gnome.desktop.a11y.interface high-contrast` is true, the generator
   writes marked, palette-free user stylesheets and does not switch the theme.
8. **Resource reading prefers PyGObject.** `gi` can load either Gtk 3.0 or
   Gtk 4.0 per process, never both, so each lookup runs in a **child
   interpreter** (`sys.executable -c …`) that loads its own Gtk namespace and
   writes to a `mkstemp` file (stdout could be polluted by a user's
   `sitecustomize`/`usercustomize`). `gresource extract` is only a secondary
   fallback and must never be invoked unguarded — `gresource` is not on every
   system (`libglib2.0-bin` on Debian-style layouts), and its absence used to
   produce a `FileNotFoundError` traceback that defeated the advertised
   PyGObject fallback. `Gio.Resource.load(libfile)` is *not* a substitute: it
   only reads standalone `.gresource` files.
9. **Markers, atomic writes, foreign-file safety.** Every generated file starts
   with `Generated by Gtk-Omarchy-Theme-Inheritar (omarchy-adwaita-gtk)`.
   `LEGACY_MARKERS` recognises the pre-rename marker so upgrades are not treated
   as foreign. Competing files are backed up (`.pre-gtk-omarchy-theme-inheritar`
   for installer-managed files, `BACKUP_SUFFIX` inside the generator) and never
   overwritten silently.
10. **nautilus-python is mandatory.** `install.sh` aborts before touching any
    file when it is missing, telling the user `sudo pacman -S nautilus-python`.
    The installer never runs `sudo` or installs packages.
11. **Never modify `/usr`.** The Nautilus extension goes to
    `~/.local/share/nautilus-python/extensions/` (nautilus-python searches
    `XDG_DATA_DIRS`), and an existing system copy from `paint-omarchy-nautilus`
    is reused rather than duplicated.

## Hard rules for changes

- No hard-coded colours or theme names; everything from `omarchy-theme-color`.
- No `GTK_THEME` in any launcher, hook or script.
- No per-application CSS (no `.gnome-disks`, `.lutris-*`, `#disks-*`, …).
- Palette-only declarations in the GTK3 user layer: never padding, margin,
  size/min-size, border-width/style, radius, font or shadow properties.
- Do not write `~/.config/gtk-3.0/gtk.css` / `gtk-4.0/gtk.css` outside the
  generator, and never without a marker and atomic write.
- Keep install/uninstall idempotent; removal must be marker- or state-driven.
- Do not commit generated CSS, `colors.toml`, `state.json`, backups or
  machine-specific paths — `.gitignore` covers these.
- Keep `hooks/60` and `hooks/70` byte-compatible with upstream (attribution in
  `LICENSE` and file headers); do not fork their behaviour for convenience.

## Known limitations / edge cases

- **Flatpak apps** never see `~/.config/gtk-*/gtk.css` (their config lives in
  `~/.var/app/<id>/config`) — out of scope.
- Apps with private CSS or colours hard-coded in C keep those colours (parts of
  GNOME Disks' own `gdu.css`-driven widgets, Electron/Chromium/Firefox, games).
- libadwaita only exposes semantic variables; widgets drawn with non-semantic
  colours keep their own values.
- The generated sheets are copies of the installed GTK stylesheets: after a GTK
  upgrade they should be regenerated (any `omarchy theme set` does it).
- `#!/usr/bin/env python3` means a non-system Python first in `PATH` (conda,
  pyenv) can shadow the interpreter and break `import gi`; the PyGObject path
  then fails and the gresource fallback must still work. Keep both paths.
- A session without a settings daemon falls back to
  `~/.config/gtk-3.0/settings.ini`; the generator only selects themes through
  `gsettings` and must degrade gracefully when it is unavailable.
- Wayland/Hyprland is the tested environment; `hyprctl` and `grim` are used by
  the manual test recipes below.

## Testing procedures

Reproduce the environment matrix rather than trusting a single run:

```bash
# 1. normal generation
/usr/bin/python3 bin/omarchy-adwaita-gtk --quiet && echo ok

# 2. PyGObject fallback: hide gresource with a symlink farm (no gresource)
FARM=/tmp/nogres-farm; rm -rf $FARM; mkdir -p $FARM
for f in /usr/bin/*; do ln -sf "$f" "$FARM/$(basename "$f")"; done; rm -f $FARM/gresource
env PATH=$FARM /usr/bin/python3 bin/omarchy-adwaita-gtk --quiet && echo ok

# 3. neither available: also shadow PyGObject
mkdir -p /tmp/nogi/gi && echo 'raise ImportError("simulated")' > /tmp/nogi/gi/__init__.py
env PATH=$FARM PYTHONPATH=/tmp/nogi /usr/bin/python3 bin/omarchy-adwaita-gtk
#    must print one actionable message, never a traceback

# 4. byte-identity: resource payloads must equal `gresource extract`
gd=$(gresource extract /usr/lib/libgtk-4.so.1 \
     /org/gtk/libgtk/theme/Default/Default-dark.css | sha256sum)
#    compare with load_resource() output from the generator for the same path

# 5. theme switching (all five artifacts must follow)
omarchy-theme-set "Catppuccin Latte"
grep -m1 theme_bg_color  ~/.config/gtk-3.0/gtk.css     # light values
grep -m1 window_bg_color ~/.config/gtk-4.0/gtk.css
grep -m1 window_bg_color ~/.cache/omarchy/gtk/nautilus.css
omarchy-theme-set "Nord Zelda"

# 6. HighContrast guard
gsettings set org.gnome.desktop.interface gtk-theme HighContrast
~/.local/bin/omarchy-adwaita-gtk --quiet
sed -n 2p ~/.config/gtk-3.0/gtk.css        # "palette not applied"
gsettings set org.gnome.desktop.interface gtk-theme Omarchy-Adwaita

# 7. installer / uninstaller
./install.sh && ./install.sh    # second run must report "already present"
./uninstall.sh                  # must leave hooks 60/70, nautilus.css, Lutris
```

Visual checks used throughout this project (window is focused first so the
compositor does not dim it):

```bash
geo=$(hyprctl clients -j | jq -r '.[]|select(.class=="org.gnome.DiskUtility")
      |"\(.at[0]) \(.at[1]) \(.size[0]) \(.size[1])"')
set -- $geo; hyprctl dispatch 'hl.dsp.focus({class="org.gnome.DiskUtility"})'
sleep 2; grim -g "$1,$2 ${3}x${4}" /tmp/disks.png
# then sample pixels with PIL: headerbar band rows 6..42, content below row 80
```

What "correct" looks like: Disks headerbar and content show the Omarchy
`darker_background` gradient and window/base colours; checked checkboxes and
radios contain bright indicator pixels with **zero** broken-image red
(`#ff0101`); Nautilus list surfaces show its own `view_bg` (`#282c34` on Nord
Zelda) rather than the generic layer's `dark_background` (`#1e2127`); Lutris
keeps its layout (window width == Atspi frame width, no clipped right edge).

## Installer / uninstaller behaviour

- `install.sh`: environment checks (Omarchy, `omarchy-theme-color`, GTK versions,
  **nautilus-python**) → `install_file` for generator/hooks (identical →
  keep, ours → update, foreign → back up and keep) → Nautilus extension
  (skipped when a system copy exists) → generate → run hooks 60/70 →
  write `install.json`. Never touches `/usr`.
- `uninstall.sh`: runs `omarchy-adwaita-gtk --restore` (removes theme dir, both
  user stylesheets, generator state, hook 80, restores the previous
  `gtk-theme`), then removes only marker/state-owned files, leaves shared hooks
  60/70, `nautilus.css`, the packaged extension, Lutris and unrelated config
  alone, and deletes the state directory.

## Environment this was built and verified against

Omarchy 4.x on Arch/Hyprland: GTK3 3.24.52, GTK4 4.22.4, libadwaita 1.9.3,
libhandy 1.8.3, Nautilus 50.3.1, nautilus-python 4.1.0; apps exercised: GNOME
Disks, handy-1-demo, Evince, Nautilus, Lutris, a libadwaita probe and a plain
GTK4 demo. The live installation is `~/.local/bin/omarchy-adwaita-gtk` plus the
hooks in `~/.config/omarchy/hooks/theme-set.d/`; after changing `bin/` run
`./install.sh` (or copy the file) so the live copy matches the repo.

## Git conventions

Imperative, sentence-case subjects (`Make nautilus-python a mandatory
dependency`, `Fix GTK resource fallback when gresource is unavailable`); one
logical change per commit; screenshots may be committed under
`docs/screenshots/`. Let the maintainer review before committing larger changes
— this project's history is deliberately granular.
