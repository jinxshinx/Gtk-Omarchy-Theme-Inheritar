# Nautilus integration

Nautilus is not an exception to be tolerated - it is part of the product. It
keeps its own specialized Omarchy palette, which is why the generic layers are
never allowed to paint over it.

## How it works

```
omarchy theme set
   │
   ├── 60-omarchy-gtk-accent.sh      → GSettings org.gnome.desktop.interface accent-color
   │                                    (libadwaita hot-reloads this everywhere)
   └── 70-omarchy-nautilus-palette.sh → ~/.cache/omarchy/gtk/nautilus.css (atomic write)
                                              │
                                              ▼
                              nautilus/omarchy_palette.py (inside Nautilus)
                              Gtk.CssProvider at Gtk.STYLE_PROVIDER_PRIORITY_USER + 1 = 801
                              + Gio directory monitor for live reload
```

The extension registers its provider at **801**, one step above the generic
GTK4 user stylesheet at **800**. In GTK the provider priority is compared before
selector specificity, so Nautilus always resolves its own values - the generic
libadwaita palette never wins inside Nautilus.

Live reload: the hook replaces `nautilus.css` atomically (write + rename). The
extension monitors the *directory* (not the file, so the rename survives),
debounces 80 ms and calls `load_from_file()` on the existing provider, which
restyles every open Nautilus window immediately.

## Where the extension is installed

nautilus-python loads extensions from every `XDG_DATA_DIRS` entry:

* `/usr/share/nautilus-python/extensions/` (system packages)
* `~/.local/share/nautilus-python/extensions/` (user-local - verified working on
  Nautilus 4x with nautilus-python 4.1)

The installer prefers the user-local path and therefore needs no root access.
If a system copy already exists (for example from the
[`paint-omarchy-nautilus`](https://github.com/JJDizz1L/paint-omarchy-nautilus)
package, which is MIT-licensed and ships exactly these hooks plus this
extension), the installer deliberately does **not** duplicate it: it reports
that the system package already provides it. Two copies would both register a
provider and reload the same file, which is pointless rather than harmful - but
duplicating a packaged file is still the wrong thing to do.

## Interaction with paint-omarchy-nautilus

`paint-omarchy-nautilus` installs its hooks through Omarchy's own hook system
(`omarchy hook install theme-set <file>`) and bootstraps them once per user from
a systemd user service. This project installs the same hooks (MIT, attributed)
into `~/.config/omarchy/hooks/theme-set.d/` itself, so it works on systems where
that package is absent. When both are present the files are byte-identical and
the installer keeps whichever is already there; the hooks are idempotent, so a
theme switch simply regenerates the same CSS once per hook.

## Uninstalling

`uninstall.sh` removes the user-local extension only when this project installed
it and no system package provides one. Hooks 60/70 are removed only when this
project installed them (tracked in
`~/.local/state/gtk-omarchy-theme-inheritar/install.json`) and the upstream
package's copy is not present.

## Verifying that Nautilus is not overridden

```bash
# generic layer says view_bg = dark_background
grep -m1 '@define-color view_bg_color' ~/.config/gtk-4.0/gtk.css
# Nautilus layer says view_bg = background (different value on purpose)
grep -m1 '@define-color view_bg_color' ~/.cache/omarchy/gtk/nautilus.css

# Nautilus renders its own value: open Nautilus and sample the list area,
# or check that the extension loaded:
nautilus --gapplication-service &   # then look for "[omarchy-palette]" in its output
```
