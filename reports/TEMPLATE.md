<!--
Copy this file into reports/inbox/ and name it YYYY-MM-DD-<platform>-<short-slug>.md,
for example 2026-12-03-gtk4-listview-scroll-jump.md. One gotcha per file.
Fill in every section. Write "Unknown" rather than leaving a section out.
Delete these comments when you're done.
See docs/WRITING-REPORTS.md for the full instructions.
-->

# <One-line title that names the trap, e.g. "`Gtk.ListView` jumps to the top when its model is replaced">

- **Date encountered:** YYYY-MM-DD
- **Reported by:** <Human name or handle, and/or the AI tool and model, e.g. "Claude Opus 5.5">
- **Project visibility:** <Public | Non-public>
- **Project:** <Public: the repo name, e.g. `qdvc-tadaima`. Non-public: write exactly "Non-public QDVC project".>
- **Platform:** <Section of the Cursed Knowledge document this belongs in, e.g. "GTK 4 and libadwaita">
- **Versions:** <OS and version, toolkit/SDK/library versions, compiler or toolchain, device if relevant>
- **Log link:** <Public: a commit-pinned permalink to the log turn where the problem was raised. Non-public: write "Not available (non-public project)".>
- **Status:** <Confirmed fixed | Fix not yet confirmed | Workaround only | Unresolved>
- **Related entry:** <Title of an existing Cursed Knowledge entry this adds to, or "None">

## Symptom

<What was seen, as close as possible to how it was first reported. Include the exact error message or warning text. Non-public projects: replace project-specific names in it with neutral placeholders such as `<MyView>`, and keep the platform's own wording intact.>

## Cause

<Why it happened. Say how sure you are: confirmed by measurement, by reading the library source, or best guess.>

## Fix

<What was done about it, and whether it was confirmed to work. Mention anything tried first that didn't work, if it was a tempting dead end.>

## Watch for

<How to avoid it next time, or where else the same trap could appear. Write "Nothing further" if there's nothing useful to add.>

## Minimal example

<Optional for public projects, strongly encouraged for non-public ones: a few lines of generic code that show the problem, with neutral names. Write "None" if not applicable.>
