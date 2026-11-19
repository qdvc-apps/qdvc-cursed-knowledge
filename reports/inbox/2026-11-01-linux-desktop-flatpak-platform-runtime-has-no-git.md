# The GNOME Flatpak `.Platform` runtime has no `git`, even though builds see it

- **Date encountered:** 2026-11-01 (month only)
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Linux desktop integration and packaging
- **Versions:** GNOME runtime (`org.gnome.Platform` / `org.gnome.Sdk`), reported against several versions; not tested in this project.
- **Log link:** Not available (non-public project)
- **Status:** Workaround only (found while planning; never run in this project)
- **Related entry:** None

## Symptom

Not hit directly. While planning a Flatpak app, research into which command-line tools are available at run time found that apps running `git` fail with "command not found", although everything worked at build time.

## Cause

`git` is in the `.Sdk` runtime (`org.gnome.Sdk`), which `flatpak-builder` uses for building, but not in the `.Platform` runtime (`org.gnome.Platform`) that installed apps run against. Source: a GNOME issue tracker report about another app's Flatpak, and the runtime's documented split between `.Sdk` and `.Platform`. Not tested in this project.

## Fix

Either build `git` into the Flatpak as an extra module in the manifest, or avoid needing the `git` binary at run time. Running the app against the SDK (`flatpak run --runtime=org.gnome.Sdk//<version> <app-id>`) hides the problem during development but isn't a fix.

## Watch for

Any command-line tool an app runs at run time: if it's only in the `.Sdk`, it won't be there for users. Test the installed app, not just `flatpak-builder --run`.

## Minimal example

```python
import subprocess
subprocess.run(["git", "--version"], check=True)
# Inside an app running on org.gnome.Platform: FileNotFoundError: [Errno 2] No such file or directory: 'git'
```
