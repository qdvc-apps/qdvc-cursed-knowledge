# QDVC Cursed Knowledge

Things we learnt the hard way while building the [QDVC apps](https://github.com/orgs/qdvc-apps/repositories), and would rather not learn again. In the spirit of [Immich's Cursed Knowledge](https://immich.app/cursed-knowledge).

Every entry here comes from a real build error, glitch, bug, defect or surprise, either recorded in the AI assistance logs of the QDVC repositories (kept according to the [Vibecoding Recordkeeping Specification](https://github.com/blairwang-online/vibecoding-recordkeeping-specification)) or sent in as a [report](reports/) from a QDVC project, public or not. The aim is practical: when a new project starts on a platform we have already used, skim the relevant section first, so the same gotchas don't have to be rediscovered and fixed from scratch.

## How to read this document

Entries are grouped by platform. Each one has the same parts:

- **Symptom:** what we saw, in roughly the words it was reported.
- **Cause:** why it happened, as diagnosed in the log.
- **Fix:** what was done about it, and whether the log shows it was confirmed to work.
- **Watch for:** how to avoid it next time (only where there is something useful to say).

The links under each title point at the exact turn in the chat log where the problem was raised, pinned to a commit so they won't drift (`UR.N` is "User request N", following the recordkeeping specification). Where a fix took several rounds, the first and last relevant turns are linked. Entries from non-public projects can't link to their logs, so they link to the report they came from in [`reports/processed/`](reports/processed/) instead. Entries marked "Non-public QDVC project" don't all come from the same project.

A word of caution: most of these diagnoses were made by an AI assistant that usually could not compile or run the app it was working on. Where the log shows the user confirming that a fix worked, the entry says so. Where it doesn't, treat the cause as the best available explanation rather than established fact.

## Contents

- [Python and GTK 3 (PyGObject)](#python-and-gtk-3-pygobject)
- [GTK 4 and libadwaita](#gtk-4-and-libadwaita)
- [Linux desktop integration and packaging](#linux-desktop-integration-and-packaging)
- [Python, the shell and the terminal](#python-the-shell-and-the-terminal)
- [Git](#git)
- [Swift, SwiftUI and AppKit (macOS)](#swift-swiftui-and-appkit-macos)
- [macOS toolchain: Xcode and SwiftPM](#macos-toolchain-xcode-and-swiftpm)
- [Swift, SwiftUI and UIKit (iOS and iPadOS)](#swift-swiftui-and-uikit-ios-and-ipados)
- [Kotlin, Jetpack Compose and Android UI](#kotlin-jetpack-compose-and-android-ui)
- [Android storage and data: SAF and Room](#android-storage-and-data-saf-and-room)
- [Android platform: permissions, networking and background work](#android-platform-permissions-networking-and-background-work)
- [Android build tooling: Gradle and Kotlin](#android-build-tooling-gradle-and-kotlin)
- [Windows, .NET and WinForms](#windows-net-and-winforms)
- [Web: HTML, CSS, SVG, JavaScript and Plotly](#web-html-css-svg-javascript-and-plotly)
- [Other libraries and services](#other-libraries-and-services): llama.cpp, Textual, WeasyPrint, Werkzeug, matplotlib, Immich
- [File formats and standards](#file-formats-and-standards)
- [Mistakes the AI kept repeating](#mistakes-the-ai-kept-repeating)
- [Sources](#sources)
- [Reporting new cursed knowledge](#reporting-new-cursed-knowledge)

## Python and GTK 3 (PyGObject)

### Assigning `self.window` in a `Gtk.Dialog` subclass crashes

[`qdvc-nice-mail` UR.7](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#7-user-request)

**Symptom:** Opening Edit → Preferences crashed with `RuntimeError: field is not writable`.

**Cause:** `Gtk.Dialog` inherits from `Gtk.Window`, which has a read-only GObject field called `window` (the underlying `GdkWindow`). The dialog's constructor stored a reference to the main window as `self.window`, which collides with that field.

**Fix:** Rename the attribute (here to `self.main_window`).

**Watch for:** Any subclass of a GTK widget. Before naming an attribute after something generic (`window`, `parent`, `name`), check it isn't already a field or property on the widget.

### Hand-built menu items break the menubar: odd left padding, no shortcut labels

[`qdvc-logbook` UR.9](https://github.com/qdvc-apps/qdvc-logbook/blob/26582b43595b457c7a1eee2f94cd1013c6907113/vibe-coding/2026-07-12-claude.md#9-user-request) · [`qdvc-nice-mail` UR.7](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#7-user-request) → [`qdvc-nice-mail` UR.15](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#15-user-request)

**Symptom:** Menubar items had a strange gap on the left, and keyboard shortcuts weren't shown next to menu items. A fixed-width icon slot made the labels line up but didn't make it look right.

**Cause:** Each item was a plain `Gtk.MenuItem` containing a custom `Box(Image + Label)`, which the QDVC GTK spec had required in order to avoid the deprecated `Gtk.ImageMenuItem`. GTK reserves its own indentation for icons and check marks, so the custom box doubles it, and a custom child also never renders the accelerator label.

**Fix:** Use the idiomatic GTK 3 menubar: `Gtk.ImageMenuItem` for items with icons, `new_with_mnemonic("_File")` for menus and items, and `add_accelerator` with an accel group so GTK draws the shortcuts itself. This also gives Alt+letter access for free. The user confirmed it worked, and the [Python GTK app specification](https://github.com/qdvc-apps/qdvc-python-gtk-app-specification) was updated afterwards to make avoiding deprecated GTK 3 widgets an explicit non-goal ([`qdvc-python-gtk-app-specification` UR.9](https://github.com/qdvc-apps/qdvc-python-gtk-app-specification/blob/7cdbc47837e9f28c3745e4c4982142c35bb2dca6/vibe-coding/2026-07-13-claude.md#9-user-request)).

**Watch for:** In a GTK 3 app aimed at MATE users, "deprecated" is not a reason to avoid the widget that gives the native look. `Gtk.ImageMenuItem` and `GtkUIManager`/`GtkAction` were both deprecated in GTK 3.10, GTK 3.24 is a frozen bug-fix series, and MATE's own Caja still builds its menus with the deprecated stack (with deprecation warnings suppressed). See [`qdvc-nice-mail` UR.19](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#19-user-request) and [`qdvc-nice-mail` UR.23](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#23-user-request).

### The About icon is blank on some systems

[`qdvc-nice-mail` UR.3](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#3-user-request) → [`qdvc-nice-mail` UR.7](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#7-user-request)

**Symptom:** Help → About showed no icon, on a machine with no custom icon set.

**Cause:** The themed icon name `help-about` isn't present in every icon theme, and the first fallback (`help-browser`) wasn't either.

**Fix:** Fall back through a chain that ends in a name from the freedesktop standard set: `help-about` → `help-browser` → `help-contents` → `dialog-information`.

**Watch for:** Any themed icon name. Look it up with a fallback rather than assuming it exists.

### `Gtk.CellRendererText` has no `get_style_context()`

[`qdvc-household-records` UR.17](https://github.com/qdvc-apps/qdvc-household-records/blob/1815df405bd8a02963b195c06945a22a4248ad19/vibe-coding/2026-07-16-claude.md#17-user-request)

**Symptom:** `AttributeError: 'CellRendererText' object has no attribute 'get_style_context'` when trying to dim a count in a tree view.

**Cause:** Cell renderers aren't widgets, so they have no style context.

**Fix:** Set the renderer's `foreground` property (or `foreground-rgba`) directly.

### Rich-text copy with `Gtk.Clipboard.set_with_data` aborts the app

[`qdvc-bibliotheca` UR.5](https://github.com/qdvc-apps/qdvc-bibliotheca/blob/f80eca2a733e82476cde34835004c27f8ebe47d3/vibe-coding/2026-07-08-claude.md#5-user-request)

**Symptom:** Copying a reference as rich text raised `AttributeError: 'Clipboard' object has no attribute 'set_with_data'`, followed by a `free(): invalid pointer` abort.

**Cause:** `set_with_data` isn't introspectable, so PyGObject doesn't expose it. The abort came from clipboard callbacks whose owner objects had been garbage-collected while GTK still held pointers to them.

**Fix:** Use `set_with_owner` with `Gtk.TargetEntry` targets, keep the owner object alive through a module-level strong reference, and fall back to plain text if that fails. Don't call `store()` on the rich-text path, since forcing a custom `text/html` target to be serialised upsets some clipboard managers.

**Watch for:** Any GTK API that takes callbacks plus user data. In Python, keep a strong reference to whatever GTK will call back into.

### `Gtk.Notebook`'s `switch-page` fires before the current page changes

[`qdvc-markdown-notebook` UR.23](https://github.com/qdvc-apps/qdvc-markdown-notebook/blob/830b62c6b69c858a2189854e867885c783a6d522/vibe-coding/vibe-coding-logs/2026-06-30-claude.md#23-user-request)

**Symptom:** With two notes open in tabs, the outline panel showed the headings of the previous tab, one switch behind.

**Cause:** `switch-page` is emitted before `Gtk.Notebook` updates its own current page, so calling `get_current_page()` inside the handler returns the old page.

**Fix:** Use the `page_num` argument the signal passes to the handler instead of asking the notebook.

### A label won't wrap even with a size request

[`qdvc-display-toggle` UR.11](https://github.com/qdvc-apps/qdvc-display-toggle/blob/870d5c70bd3a8fa5145d5557e258998c8f3b0d01/vibe-coding/2026-06-29-claude.md#11-user-request) → [`qdvc-display-toggle` UR.15](https://github.com/qdvc-apps/qdvc-display-toggle/blob/870d5c70bd3a8fa5145d5557e258998c8f3b0d01/vibe-coding/2026-06-29-claude.md#15-user-request)

**Symptom:** A header label stayed on one long line no matter what pixel width was requested.

**Cause:** `set_size_request` only sets a minimum. The label still advertised its full single-line natural width, and because its parent box stretched to the window width, it never needed to wrap.

**Fix:** Use `set_max_width_chars(N)` so the label reports a small preferred width, and centre the parent (`halign=CENTER`) so it doesn't stretch. The two settings only work together.

### `Gtk.Statusbar.push()` shows Pango markup literally

[`qdvc-display-toggle` UR.19](https://github.com/qdvc-apps/qdvc-display-toggle/blob/870d5c70bd3a8fa5145d5557e258998c8f3b0d01/vibe-coding/2026-06-29-claude.md#19-user-request)

**Symptom:** Italic `<i>` tags pushed to the status bar appeared as literal text.

**Cause:** `push()` sets the text with `set_text()`, not `set_markup()`.

**Fix:** Reach into the status bar's message area (`get_message_area().get_children()[0]`, the internal `Gtk.Label`) and call `set_markup()` on it. This relies on GTK internals; it works on GTK 3.24 but is the first thing to check after a GTK upgrade.

### A `POPUP` window doesn't reliably get keyboard focus

[`qdvc-display-toggle` UR.3](https://github.com/qdvc-apps/qdvc-display-toggle/blob/870d5c70bd3a8fa5145d5557e258998c8f3b0d01/vibe-coding/2026-06-29-claude.md#3-user-request)

**Symptom:** In a small popup chooser, the arrow keys did nothing.

**Cause:** Windows created with `Gtk.WindowType.POPUP` bypass the window manager and don't reliably receive keyboard focus.

**Fix:** Use a `TOPLEVEL` window and call `present()`. This also gives a normal titlebar.

### A toolbar button shows a focus ring on startup

[`qdvc-markdown-notebook` UR.47](https://github.com/qdvc-apps/qdvc-markdown-notebook/blob/830b62c6b69c858a2189854e867885c783a6d522/vibe-coding/vibe-coding-logs/2026-06-28-claude.md#47-user-request)

**Symptom:** The New note toolbar button looked highlighted as soon as the app launched.

**Cause:** It was the first focusable widget, so it got keyboard focus and its focus ring.

**Fix:** `set_can_focus(False)` on toolbar buttons, which matches how MATE toolbars behave.

### In "beside" toolbar style, only "important" items show labels

[`qdvc-markdown-notebook` UR.51](https://github.com/qdvc-apps/qdvc-markdown-notebook/blob/830b62c6b69c858a2189854e867885c783a6d522/vibe-coding/vibe-coding-logs/2026-06-28-claude.md#51-user-request)

**Symptom:** Switching the toolbar to icons-with-labels-beside showed labels on some buttons and not others.

**Cause:** With `Gtk.ToolbarStyle.BOTH_HORIZ`, GTK only shows the label of tool items marked `set_is_important(True)`.

**Fix:** Mark the buttons whose labels should appear as important.

### Right-clicking a tree view row selects it

[`qdvc-markdown-notebook` UR.11](https://github.com/qdvc-apps/qdvc-markdown-notebook/blob/830b62c6b69c858a2189854e867885c783a6d522/vibe-coding/vibe-coding-logs/2026-06-30-claude.md#11-user-request)

**Symptom:** Right-clicking a note in the list (to open its context menu) also selected and opened it.

**Cause:** The tree view's default button-press handler selects the row for any button.

**Fix:** In the `button-press-event` handler, show the menu and return `True` for right-clicks, so the default handler doesn't run.

### Clearing and refilling a store loses the selection and fires handlers

[`qdvc-equip` UR.1](https://github.com/qdvc-apps/qdvc-equip/blob/8a7e8511e54e3473028ee5c4c0143fad3cf77c80/vibe-coding/vibe-coding-logs/2026-07-02-claude.md#1-user-request)

**Symptom:** Saving an asset changed what was selected in the panes, and opened things the user hadn't clicked.

**Cause:** The refresh called `store.clear()` and repopulated it. Clearing drops the selection and emits selection-changed signals, which triggered handlers that opened other items.

**Fix:** Update the affected row in place instead of clearing and repopulating.

**Watch for:** Any "refresh the list" code path. In GTK 4 the same trap exists with `Gio.ListStore.remove_all()` (see the expansion entry under GTK 4).

### Running a package's script from inside its own folder breaks relative imports

[`qdvc-countdowns` UR.3](https://github.com/qdvc-apps/qdvc-countdowns/blob/53d548a00a744d18eba6b025dea1c20f0fd031cd/vibe-coding/2026-07-11-claude.md#3-user-request)

**Symptom:** `ImportError: attempted relative import beyond top-level package`.

**Cause:** Launching the entry script from inside its own directory makes the library folder a top-level package with no parent, so `from .. import` has nowhere to go.

**Fix:** Import from the package by name (absolute imports) so it works however it's launched. `python3 -m package.module` from the parent directory is a quick workaround.

### `gtk.css: Junk at end of value for border-radius` isn't your bug

[`qdvc-household-records` UR.25](https://github.com/qdvc-apps/qdvc-household-records/blob/1815df405bd8a02963b195c06945a22a4248ad19/vibe-coding/2026-07-16-claude.md#25-user-request)

**Symptom:** A `Gtk-WARNING` about `gtk.css` appeared in a traceback next to a real crash.

**Cause:** The app shipped no CSS; the warning came from the system GTK theme's stylesheet. The real crash was an unrelated missing `import os`.

**Fix:** None needed in the app. Don't let theme warnings send you on a detour.

## GTK 4 and libadwaita

### `GLib.idle_add(widget.grab_focus)` spins a CPU core forever

[`qdvc-tadaima` UR.33](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#33-user-request) → [`qdvc-tadaima` UR.55](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#55-user-request)

**Symptom:** After the photo viewer had been opened once, the app sat at 100% of one core (about 13% of an 8-core machine) indefinitely, even when idle and back in the gallery.

**Cause:** GLib re-runs an idle callback for as long as it returns a truthy value. `grab_focus()` returns `True` on success, so passing it directly to `idle_add` re-arms it on every main-loop iteration. Python profilers showed 0% in app code because the spinning happens in C inside `Gio.Application.run`; only `py-spy dump --native` revealed the GObject-introspection callback being dispatched over and over.

**Fix:** Wrap it so the callback returns `False`: `GLib.idle_add(lambda: (widget.grab_focus(), False)[1])`, or a small helper. Audit every `idle_add` and `timeout_add` that passes a method you didn't write. Confirmed fixed in [`qdvc-tadaima` UR.57](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#57-user-request).

**Watch for:** The debugging took about 12 rounds of theorising about rendering, the filmstrip, animations and I/O before measurement found it. The first CPU probe written for the investigation had the same bug (a self-re-arming idle callback) and caused its own spin. The full story, with the isolation tests that worked, is in [`the-idle-callback-that-never-slept.md`](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/qdvc-tadaima-gtk4/docs/2026-07-25-the-idle-callback-that-never-slept.md).

### A `Gtk.Picture` loaded from a file reports the full image size

[`qdvc-tadaima` UR.15](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#15-user-request) → [`qdvc-tadaima` UR.25](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#25-user-request)

**Symptom:** The thumbnail grid showed one column, or three columns with some folders' thumbnails taking a third of the width and others tiny. This persisted over about five rounds of `Gtk.FlowBox` tuning.

**Cause:** A `Gtk.Picture` created from a file has the source image's pixel dimensions as its natural size, regardless of `can_shrink` or `content_fit`. `set_size_request` only sets a minimum, and GTK boxes have no maximum, so the huge natural size leaked into `FlowBox`'s layout, which computes columns from the widest child. (A separate round found the detail area wasn't set to `hexpand`, which caused the "invisible right panel".)

**Fix:** Load the thumbnail into a `GdkPixbuf`, scale it once to fit the slot, wrap it in a `Gdk.Texture`, and give that to the picture. Its natural size is then the bounded size.

**Watch for:** Any grid or list of images in GTK 4. Also: a CSS `outline` on a `Gtk.Picture` was unreliable for showing selection; a `box-shadow` ring worked.

### Rebuilding a `Gtk.TreeListModel`'s root store collapses everything

[`qdvc-tadaima` UR.11](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#11-user-request)

**Symptom:** While a background scan was running, the sidebar kept "retreating" to the parent folder whenever the user expanded a subfolder.

**Cause:** A periodic live refresh called `remove_all()` on the root store and repopulated it, which destroyed the tree rows and their expansion state.

**Fix:** Don't touch the tree store during a scan; refresh only the grid. Persist and reapply expansion state deliberately when the tree really must be rebuilt.

### Filling a `FlowBox` with hundreds of children freezes the window

[`qdvc-tadaima` UR.3](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-10-16-claude.md#3-user-request)

**Symptom:** Clicking a folder with 300+ photos froze the UI for a time proportional to the number of photos.

**Cause:** Every child was created and every thumbnail decoded synchronously on the main thread.

**Fix:** Add correctly sized placeholder slots a few at a time from idle callbacks (sizes come from the index, so nothing has to be decoded first), decode thumbnails on background threads, cancel pending work when the folder changes, and update the open grid in place during scans instead of rebuilding it.

### `Adw.SpinRow` with a live `notify::value` handler crashes

[`qdvc-household-records` UR.7](https://github.com/qdvc-apps/qdvc-household-records/blob/1815df405bd8a02963b195c06945a22a4248ad19/vibe-coding/2026-07-16-claude.md#7-user-request)

**Symptom:** Toggling the periodic setting on an account crashed the GTK 4 build.

**Cause:** An `Adw.SpinRow` created with `new_with_range` had a `notify::value` handler that fired during `set_value`, while the row was still being set up.

**Fix:** Move the settings into an on-demand dialog with no live binding. Generally, connect value-change handlers after setting initial values, or guard them.

### Deprecated libadwaita dialogs, and no frozen rows in GTK's table

[`qdvc-progress-gtk4` UR.3](https://github.com/qdvc-apps/qdvc-progress-gtk4/blob/52abc25fb53a95a259e7d0c0e719d559053bdee6/ai-assistance-logs/2026-10-14-claude.md#3-user-request)

**Symptom:** Not a crash: the QDVC spec named window types that current libadwaita has deprecated, and the app needed spreadsheet-like frozen rows and columns.

**Cause:** libadwaita 1.5 replaced the older window-based dialogs (such as `Adw.PreferencesWindow`) with dialog widgets (`Adw.PreferencesDialog`). GTK's standard table widget can't freeze rows or columns or select rectangles of cells.

**Fix:** Use `Adw.PreferencesDialog` and friends (so the app needs libadwaita 1.5+, i.e. GNOME 46 / Ubuntu 24.04 / Fedora 40), and draw the grid as a custom widget.

### Porting colours from AppKit: `withAlphaComponent` replaces alpha

[`qdvc-progress-gtk4` UR.7](https://github.com/qdvc-apps/qdvc-progress-gtk4/blob/52abc25fb53a95a259e7d0c0e719d559053bdee6/ai-assistance-logs/2026-10-14-claude.md#7-user-request)

**Symptom:** The GTK 4 port's grid lines and hatching looked far lighter than the Mac app's, even after one correction.

**Cause:** AppKit's `withAlphaComponent(0.35)` sets the colour's alpha to 35% outright; it doesn't scale an existing alpha. The port had taken "35% separator" to mean 35% of an already pale separator colour, so its lines were about a third as strong.

**Fix:** Use the effective values (35% and 50% foreground on the pane colour) as named constants, and measure pixel values of a side-by-side render rather than eyeballing.

## Linux desktop integration and packaging

### The MATE panel shows a generic icon and doesn't match the launcher

[`qdvc-bibliotheca` UR.23](https://github.com/qdvc-apps/qdvc-bibliotheca/blob/f80eca2a733e82476cde34835004c27f8ebe47d3/vibe-coding/2026-07-08-claude.md#23-user-request) → [`qdvc-bibliotheca` UR.25](https://github.com/qdvc-apps/qdvc-bibliotheca/blob/f80eca2a733e82476cde34835004c27f8ebe47d3/vibe-coding/2026-07-08-claude.md#25-user-request) · [`qdvc-countdowns` UR.7](https://github.com/qdvc-apps/qdvc-countdowns/blob/53d548a00a744d18eba6b025dea1c20f0fd031cd/vibe-coding/2026-07-11-claude.md#7-user-request)

**Symptom:** The running app appeared in the MATE panel with a generic icon, not associated with its menu launcher.

**Cause:** Three things have to line up: the process's X11 `WM_CLASS`, the `.desktop` file's `StartupWMClass`, and an icon name that actually resolves. The first attempt bundled an SVG under the app's own name, which only works if the user installs it into their icon theme.

**Fix:**

1. Call `GLib.set_prgname("app-name")` at import time, before any window is created. Setting it later has no effect once the window class is established.
2. Put `StartupWMClass=app-name` in the `.desktop` file, exactly equal to the prgname.
3. Use a standard themed icon name (for example `accessories-dictionary` or `appointment-soon`) in both the `.desktop` file and `set_default_icon_name`, rather than a bundled file.

Keep the WM class and icon name as single constants in one module. Check with `xprop WM_CLASS` on the running window, and expect to log out and back in once, because MATE caches the association per session.

### GNOME on Wayland matches windows to launchers by `.desktop` file name

[`qdvc-progress-gtk4` UR.3](https://github.com/qdvc-apps/qdvc-progress-gtk4/blob/52abc25fb53a95a259e7d0c0e719d559053bdee6/ai-assistance-logs/2026-10-14-claude.md#3-user-request)

**Symptom:** (Avoided.) The GTK 4 app would otherwise show up unmatched in GNOME's dock.

**Cause:** On Wayland there is no `WM_CLASS`; GNOME matches a window to its launcher by the application ID, which must equal the `.desktop` file's name.

**Fix:** Name the desktop entry after the application ID, for example `qdvc.Progress.desktop` for application ID `qdvc.Progress`.

### `WindowPosition.CENTER` is ignored on Wayland

[`qdvc-bibliotheca` UR.5](https://github.com/qdvc-apps/qdvc-bibliotheca/blob/f80eca2a733e82476cde34835004c27f8ebe47d3/vibe-coding/2026-07-08-claude.md#5-user-request)

**Symptom:** Centring the window on startup works on X11 (MATE) but may not under Wayland.

**Cause:** Under Wayland, the compositor decides window placement. Apps can't override it.

**Fix:** None from the app side. Don't spend rounds trying.

### `xrandr --same-as` silently fails across different resolutions

[`qdvc-display-toggle` UR.5](https://github.com/qdvc-apps/qdvc-display-toggle/blob/870d5c70bd3a8fa5145d5557e258998c8f3b0d01/vibe-coding/2026-06-29-claude.md#5-user-request)

**Symptom:** Mirror mode had no effect, from both single-display and extended mode.

**Cause:** `--auto` gives each output its own preferred mode. If the laptop panel and external monitor have different native resolutions, `--same-as` can't overlay them and does nothing visible.

**Fix:** Parse the modes each output supports, pick a resolution they share, and set it explicitly on all of them. Surface `xrandr` errors in the UI. If there's no common mode, scaled mirroring (`--scale`/`--fb`) is the remaining option.

### Matching taskbar buttons to windows through AT-SPI is fragile

[`qdvc-super-switcher` UR.5](https://github.com/qdvc-apps/qdvc-super-switcher/blob/7da8bc26dfc46e713d089cc4c595317f1e9bd6b5/vibe-coding/2026-06-29-claude.md#5-user-request)

**Symptom:** (Design limitation.) Following the user's drag-reordering of MATE's window list required reading the applet's order.

**Cause:** Through AT-SPI, task buttons expose the window title, not the X window ID, so matching them to `_NET_CLIENT_LIST` entries for `xdotool` has to go by title. It also needs accessibility switched on (`gsettings set org.mate.interface accessibility true`, then re-login) and `gir1.2-atspi-2.0`.

**Fix:** Accept the limitation, or avoid the approach. Duplicate titles (two terminals, two "Untitled Document" windows) and window grouping break the matching.

### Flatpak: the manifest was hidden by `.gitignore`, and the config moved

[`hernantz-timezones` UR.3](https://github.com/qdvc-apps/hernantz-timezones/blob/5cde5ddc0b70a8661cdfb81fcce2c7fafaf583fd/ai-assistance-logs/2026-09-29-claude.md#3-user-request)

**Symptom:** The Flatpak manifest the README referred to had never made it into the repository.

**Cause:** `.gitignore` contained `*.json`, which matched the manifest. Separately, inside the sandbox the app stores its config under `~/.var/app/<app-id>/config/`, not `~/.config/`, so existing settings don't carry over.

**Fix:** `git add -f` the manifest, or add `!packaging/flatpak/*.json` to `.gitignore`. Tell users to copy their config across once. Pick a GNOME runtime that ships the libadwaita version the app needs (1.7+ here).

### The GNOME `.Platform` runtime has no `git`, though builds see it

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-linux-desktop-flatpak-platform-runtime-has-no-git.md)

**Symptom:** (Found while planning; not tested.) A Flatpak app that runs `git` (or another command-line tool) at run time fails with "command not found" or `FileNotFoundError`, although everything worked during the build.

**Cause:** `git` is in the `.Sdk` runtime (`org.gnome.Sdk`), which `flatpak-builder` builds against, but not in the `.Platform` runtime (`org.gnome.Platform`) that installed apps run on. [Meld hit the same problem](https://gitlab.gnome.org/GNOME/meld/issues/319).

**Fix:** Build `git` into the Flatpak as an extra module in the manifest, or don't depend on the binary at run time. Running the app against the SDK (`flatpak run --runtime=org.gnome.Sdk//<version> <app-id>`) hides the problem during development but doesn't fix it.

**Watch for:** Any command-line tool an app runs at run time. If it's only in the `.Sdk`, users won't have it, so test the installed app, not just `flatpak-builder --run`.

## Python, the shell and the terminal

### oh-my-zsh ignores completions added to `fpath` after it loads

[`qdvc-gtd-eml` UR.53](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-06-23-conversation-with-claude.md#53-user-request) → [`qdvc-gtd-eml` UR.57](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-06-23-conversation-with-claude.md#57-user-request)

**Symptom:** A custom `_gtd` completion was never used; Tab fell back to filename completion.

**Cause:** Two stacked problems. First, oh-my-zsh runs `compinit` partway through `~/.zshrc`, so a `fpath=(~/.zsh/completions $fpath)` line placed after `source $ZSH/oh-my-zsh.sh` comes too late. Second, `gtd` was an alias, and zsh expands aliases before completing (unless `complete_aliases` is set), so completion saw `python3 gtd.py`, not `gtd`. `Ctrl-X h` showed the real completion context.

**Fix:** Put the completion file in `$ZSH_CUSTOM/completions` (oh-my-zsh adds it to `fpath` before `compinit`), define `gtd` as a shell function instead of an alias (`gtd() { python3 /path/to/gtd.py "$@" }`), then `rm -f ~/.zcompdump*` and `exec zsh`.

### Colours vanish when piping to `less -R`

[`qdvc-gtd-eml` UR.29](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-06-23-conversation-with-claude.md#29-user-request)

**Symptom:** `gtd.py | less -R` showed no colour.

**Cause:** The script disabled colour whenever `sys.stdout.isatty()` was false, which includes being piped to a pager.

**Fix:** Add a config or environment override to force colour (or use `unbuffer`), then pipe to `less -R` (`-RFX` is a good combination).

### `strftime("%-d")` doesn't work on Windows

[`qdvc-podcaster` UR.3](https://github.com/qdvc-apps/qdvc-podcaster/blob/284c6d4df3625f08828295ff7913bb36833e7552/ai-assistance-logs/2026-08-23-claude.md#3-user-request)

**Symptom:** (Caught before shipping.) Day-of-month without a leading zero.

**Cause:** `%-d` is a glibc extension; Windows' C runtime doesn't support it.

**Fix:** Format the number yourself, for example `str(d.day)`.

### `tempfile.mkstemp` files are mode 0600

[`qdvc-progress-gtk4` UR.1](https://github.com/qdvc-apps/qdvc-progress-gtk4/blob/52abc25fb53a95a259e7d0c0e719d559053bdee6/ai-assistance-logs/2026-10-14-claude.md#1-user-request)

**Symptom:** (Caught before shipping.) Atomically replacing a workspace file via a temporary file would have tightened its permissions.

**Cause:** `mkstemp` creates files readable only by the owner. After `os.replace`, the replaced file keeps the temp file's mode.

**Fix:** Copy the original file's mode onto the temp file before replacing it.

### Python has no standard-library TOML writer

[`qdvc-markdown-notebook` UR.7](https://github.com/qdvc-apps/qdvc-markdown-notebook/blob/830b62c6b69c858a2189854e867885c783a6d522/vibe-coding/vibe-coding-logs/2026-06-28-claude.md#7-user-request)

**Symptom:** (Design decision.) TOML looked like the natural format for a small config file.

**Cause:** `tomllib` (Python 3.11+) is read-only. Writing TOML needs `tomli-w`, which reintroduces a dependency.

**Fix:** For small dependency-free configs that the app must write back, JSON is the stdlib option that reads and writes.

### HTML in Python f-strings needs every CSS/JS brace doubled

[`qdvc-health-dashboard` UR.11](https://github.com/qdvc-apps/qdvc-health-dashboard/blob/d7fc5141a495ab0503ac9eb55e623d789cde8f04/vibe-coding/2026-07-06-claude-1.md#11-user-request) → [`qdvc-health-dashboard` UR.23](https://github.com/qdvc-apps/qdvc-health-dashboard/blob/d7fc5141a495ab0503ac9eb55e623d789cde8f04/vibe-coding/2026-07-06-claude-1.md#23-user-request)

**Symptom:** A recurring maintenance hazard in the generated dashboard: every `{` in embedded CSS or JavaScript had to be written `{{`.

**Cause:** The whole page was one big f-string, with no autoescaping of data values either.

**Fix:** Move CSS and JavaScript into plain asset files, then move the HTML into Jinja templates, which also autoescape data values.

## Git

### A plain `git pull` refuses diverged branches

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-new-git-pull-refuses-divergent-branches.md)

**Symptom:** Tools, scripts and tests that expect `git pull` to merge on its own fail when the local and remote branches have both moved on:

```
hint: You have divergent branches and need to specify how to reconcile them.
...
fatal: Need to specify how to reconcile divergent branches.
```

**Cause:** Newer Git versions (2.33.1 and later, as far as the report could establish; 2.27 to 2.33 only warned) no longer default to a merge when a pull can't fast-forward, and make the user choose. Developers rarely notice, because their own `~/.gitconfig` usually sets `pull.rebase`. It shows up on other people's machines, and in tests that isolate Git's configuration with `GIT_CONFIG_GLOBAL=/dev/null` or a fresh `HOME`.

**Fix:** Pass `--no-rebase`, `--rebase` or `--ff-only` explicitly in scripts and in any pull command shown to users. If a workflow depends on one behaviour, set `pull.rebase` in the repository's own config, which takes precedence over the user's global setting. (Workaround only.)

**Watch for:** Tests that start Git from a clean configuration behave like a new user's machine. That's useful, as long as the code under test doesn't rely on the developer's personal settings.

## Swift, SwiftUI and AppKit (macOS)

### `@Environment` model lookups crash inside separately hosted views

[`qdvc-case-manager` UR.7](https://github.com/qdvc-apps/qdvc-case-manager/blob/667ff6e8c142cd39f7c287bba5f450d48a7f8254/ai-assistance-logs/2026-11-05-claude.md#7-user-request) · [`qdvc-empirics` UR.8](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#8-user-request)

**Symptom:** `EXC_BREAKPOINT (SIGTRAP)` on the main thread, with `EnvironmentValues.subscript.getter` → `_assertionFailure` in the stack. In Case Manager it happened the moment the case table first drew rows; in Empirics, when clicking sidebar items with Back/Forward toolbar buttons present.

**Cause:** `@Environment(AppModel.self)` is a fatal error when no ancestor supplied the model. On macOS, SwiftUI hosts some views separately from the main view tree (each `Table` cell, toolbar items, popovers, inspectors), and they don't reliably inherit the environment.

**Fix:** Pass the model to those views explicitly as a parameter. Case Manager added a test that fails if a view reading the model from the environment is used in a table column.

**Watch for:** Any `@Environment` read in a `Table` column, `.toolbar`, `.popover` or `.inspector`.

### Bindings that write back unchanged values freeze the app

[`qdvc-case-manager` UR.11](https://github.com/qdvc-apps/qdvc-case-manager/blob/667ff6e8c142cd39f7c287bba5f450d48a7f8254/ai-assistance-logs/2026-11-05-claude.md#11-user-request) → [`qdvc-case-manager` UR.15](https://github.com/qdvc-apps/qdvc-case-manager/blob/667ff6e8c142cd39f7c287bba5f450d48a7f8254/ai-assistance-logs/2026-11-05-claude.md#15-user-request)

**Symptom:** The app froze (Force Quit needed) when sorting the All Cases list, when removing a filter, or when switching to that list from another page. It got worse the longer the app had been running.

**Cause:** Several things compounded. The final diagnosis was an update loop: AppKit controls (table, sidebar, search field, pickers) wrote their current values back through bindings, which triggered `@Observable` notifications, which caused redraws, which wrote back again. Earlier rounds also found work that grew over a session: each double-click opened another Decision Support window that kept redrawing, table cells carried tooltips and read the model directly, and the list was re-filtered and re-sorted on every redraw.

**Fix:** Bind controls through an equality-guarded helper that ignores writes of unchanged values, defer model observers' work to the next run-loop turn, keep table cells as pure values (no model reads, no tooltips), and reuse a single secondary window. A watchdog now logs the last 40 actions to a file if the main thread stalls for more than 3 seconds. (Not confirmed in the log.)

### The sidebar glitches into the window buttons

[`qdvc-empirics` UR.8](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#8-user-request) → [`qdvc-empirics` UR.17](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#17-user-request)

**Symptom:** Clicking most sidebar items made the sidebar rows disappear, get cut off, or push up under the close/minimise/zoom buttons. Only some pages (Project Settings, Layers, file pages) were unaffected. Resizing didn't recover it.

**Cause:** Never pinned down exactly. Partial fixes over five rounds (passing the model to toolbar items, making sidebar rows single-line so they can't be mismeasured, replacing SwiftUI `Table` with a hand-made table) didn't solve it. The pattern that finally sorted every page into "glitches" or "doesn't" was that the broken pages had a fixed header above their scrolling content.

**Fix:** Copy a sibling app that works. The window was rebuilt on [QDVC Response Letter](https://github.com/qdvc-apps/qdvc-response-letter-mac)'s structure: a `NavigationSplitView` with no column-visibility binding, a toolbar with right-hand items only (Back/Forward moved to the Go menu), the notice banner as a top inset, and every page starting with its scrolling content (a SwiftUI `Table` first, with a button bar below). Confirmed fixed in [`qdvc-empirics` UR.19](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#19-user-request).

**Watch for:** Toolbar items in `.navigation` placement on recent macOS end up in the sidebar's title-bar area beside the window buttons.

### `HSplitView` ignores SwiftUI's safe area

[`qdvc-gtd-eml-mac` UR.9](https://github.com/qdvc-apps/qdvc-gtd-eml-mac/blob/8f210e1e48dbcb84d4929f3ebe876fefde3f4064/ai-assistance-logs/2026-09-30-claude.md#9-user-request) · [`qdvc-gtd-eml-mac` UR.9](https://github.com/qdvc-apps/qdvc-gtd-eml-mac/blob/8f210e1e48dbcb84d4929f3ebe876fefde3f4064/ai-assistance-logs/2026-10-21-claude.md#9-user-request)

**Symptom:** A notification banner covered the message list's title. Later, the window tab bar cut across the list and reading pane but looked fine in single-message tabs.

**Cause:** `HSplitView` is AppKit's split view and doesn't honour SwiftUI's safe-area insets, so content in it is drawn at the very top as if the banner or the taller tab-bar title area weren't there. The banner's tinted background also extended upwards by default, behind the toolbar.

**Fix:** For banners, stack them above the content in a `VStack` instead of using a top safe-area inset, and stop the background extending. For the tab bar, split the list and reading pane with SwiftUI instead of `HSplitView`.

### A `WindowGroup` always adds a `+` button to the tab bar

[`qdvc-gtd-eml-mac` UR.9](https://github.com/qdvc-apps/qdvc-gtd-eml-mac/blob/8f210e1e48dbcb84d4929f3ebe876fefde3f4064/ai-assistance-logs/2026-10-21-claude.md#9-user-request)

**Symptom:** Message windows opened as tabs showed a meaningless `+` button, flashed as a separate window when opened, and came back at launch.

**Cause:** Windows from a SwiftUI `WindowGroup` always offer a "new window" action to the tab bar, and SwiftUI restores them.

**Fix:** Create the message windows as AppKit windows and add them to the tab group before showing them.

### AppKit text views inside SwiftUI blow up the layout

[`qdvc-bibliotheca-mac` UR.9](https://github.com/qdvc-apps/qdvc-bibliotheca-mac/blob/e5a04934b62160b8f2648f003ba87006d83edd57/ai-assistance-logs/2026-09-30-claude.md#9-user-request) → [`qdvc-bibliotheca-mac` UR.11](https://github.com/qdvc-apps/qdvc-bibliotheca-mac/blob/e5a04934b62160b8f2648f003ba87006d83edd57/ai-assistance-logs/2026-09-30-claude.md#11-user-request)

**Symptom:** Selecting a record blanked the detail pane and left the sidebar able to scroll only a couple of rows. After a first fix, clicking a record made the sidebar and list jump as if paged down, without showing the details.

**Cause:** Two separate traps that made the split view taller than the window. The notes editor wrapped an `NSScrollView` + `NSTextView` without telling SwiftUI how big it wanted to be, so SwiftUI used AppKit's fitting size, which is enormous for a scroll view. And the formatted reference used `.fixedSize(vertical: true)`: when SwiftUI probes for a column's minimum size it proposes a near-zero width, at which the text wraps one character per line and demands thousands of points of height.

**Fix:** Give wrapped AppKit views an explicit sizing rule (take exactly the space offered), and use `layoutPriority` instead of `.fixedSize(vertical:)` for text that should usually show in full.

### An `NSTextView` in an `NSScrollView` wraps wider than its pane

[`qdvc-response-letter-mac` UR.11](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#11-user-request) → [`qdvc-response-letter-mac` UR.25](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#25-user-request)

**Symptom:** Long lines ran off the right edge of the pane, so only the middle of each line was visible. A 1000-point readable column didn't help: the column was the right width but the text inside was still wider.

**Cause:** Each text view was created 400 points wide before SwiftUI had sized the surrounding scroll view (which starts at zero), and AppKit's autoresizing then kept that 400-point difference as the pane grew. Three attempts to correct the width from inside the scroll view (a starting width, re-tiling, explicit line widths) all failed.

**Fix:** Remove the AppKit scroll-view layer. Every text view sizes itself directly in SwiftUI (exactly as wide as it's given, as tall as its text) inside a SwiftUI `ScrollView`, the same way the table cells that already worked did. Confirmed fixed in [`qdvc-response-letter-mac` UR.27](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#27-user-request).

**Watch for:** If one kind of view already behaves, copy its approach before trying to repair the one that doesn't.

### A text view that measures itself during layout causes choppy scrolling

[`qdvc-response-letter-mac` UR.33](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#33-user-request)

**Symptom:** The decision letter view scrolled choppily once it had margin labels beside the text.

**Cause:** The view placed its margin labels during SwiftUI's size requests, including at a stand-in width. Moving the labels triggered new layouts, and each size request re-laid out the whole letter.

**Fix:** Cache the measured height by width and text version, place labels only when the real frame changes, and only resize a text container when the width actually changes.

### Copying from an `NSTextView` in a table cell copied nothing

[`qdvc-response-letter-mac` UR.29](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#29-user-request) → [`qdvc-response-letter-mac` UR.31](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#31-user-request)

**Symptom:** Paste into table cells worked, but Copy put nothing on the pasteboard.

**Cause:** Copy went through `NSTextView`'s `writeSelection` machinery, which had been extended with a private pasteboard type for response markup; somewhere in that path nothing was written.

**Fix:** Have Copy and Cut write to the pasteboard directly: plain text always, plus the private markup type when copying from a response. Confirmed working in [`qdvc-response-letter-mac` UR.33](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#33-user-request).

### A SwiftUI `Menu` on macOS sizes to its widest item and drops custom labels

[`qdvc-response-letter-mac` UR.9](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#9-user-request) · [`qdvc-response-letter-mac` UR.27](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#27-user-request)

**Symptom:** A page forced the whole window extremely wide. Later, a table column looked empty.

**Cause:** On macOS a SwiftUI menu button is an `NSPopUpButton`, which with `.fixedSize()` sizes itself to its widest item (here, 60-character comment excerpts). Also, a SwiftUI menu's label can only show text and SF Symbols, so a custom-drawn coloured dot was silently dropped, leaving the column apparently blank.

**Fix:** Don't use `.fixedSize()` on menus with long items. For labels that need custom drawing, use an AppKit pop-up button wrapped for SwiftUI.

### `DatePicker` can't show seconds on macOS

[`qdvc-datesetter` UR.4](https://github.com/qdvc-apps/qdvc-datesetter/blob/87f7bf4fa7352026bf25f3b79d81ae8e5503fafc/ai-assistance-logs/2026-11-06-claude.md#4-user-request)

**Symptom:** `'hourMinuteAndSecond' is unavailable in macOS`.

**Cause:** That `DatePickerComponents` option exists only on iOS and iPadOS.

**Fix:** Wrap AppKit's `NSDatePicker`, which can show seconds, for SwiftUI.

### Main-actor isolation errors in a toolbar builder struct

[`qdvc-gtd-eml-mac` UR.7](https://github.com/qdvc-apps/qdvc-gtd-eml-mac/blob/8f210e1e48dbcb84d4929f3ebe876fefde3f4064/ai-assistance-logs/2026-09-30-claude.md#7-user-request)

**Symptom:** `main actor-isolated property 'selectedRecords' can not be referenced from a nonisolated context`.

**Cause:** The toolbar was built in a plain struct, not a `View`. Views are main-actor isolated automatically; plain structs aren't, so they can't read a main-actor model.

**Fix:** Mark the struct (and the `Commands` struct) `@MainActor` and hold the model as `@Bindable`.

### A public struct's memberwise initialiser is internal

[`qdvc-response-letter-mac` UR.5](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#5-user-request)

**Symptom:** `'Renumbering' initializer is inaccessible due to 'internal' protection level`, from the app target calling into the core library.

**Cause:** Swift synthesises memberwise initialisers as `internal` even for `public` structs, so another module can't use them.

**Fix:** Write explicit `public init`s for every public struct the app needs to construct. The same round also removed `MainActor.assumeIsolated` wrappers that returned a non-`Sendable` `NSAttributedString` (a warning now, an error in Swift 6 mode).

### `private(set)` doesn't reach an extension in another file

[`qdvc-empirics` UR.4](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#4-user-request)

**Symptom:** `cannot use mutating member on immutable value: 'lostSelections' setter is inaccessible`, plus a type-checker timeout in another file.

**Cause:** Swift's `private` is file-scoped, so an extension of the same class in `AppModel+Coding.swift` can't use a `private(set)` setter declared in `AppModel.swift`. Separately, a long chain of tuple-returning closures (rank, sort, map in one expression) was too much for the type checker.

**Fix:** Use `internal` (plain `var`) or `fileprivate` deliberately. Split long closure chains into explicitly typed steps. The same timeout also comes from [literal arithmetic in SwiftUI modifier chains](#unable-to-type-check-this-expression-in-reasonable-time-from-literal-arithmetic-in-a-modifier-chain).

### Your type names can clash with SwiftUI's

[`qdvc-progress-mac` UR.7](https://github.com/qdvc-apps/qdvc-progress-mac/blob/c5e00bc3b056c1717244562044646ccdb732418a/ai-assistance-logs/2026-10-10-claude.md#7-user-request)

**Symptom:** `'GridRow' is ambiguous for type lookup in this context`, followed by a cascade of key-path errors.

**Cause:** The core library defined `GridRow`, and SwiftUI has its own `GridRow`. The app imported both.

**Fix:** Rename the core type (to `LayoutRow`). Check core type names against SwiftUI, AppKit and Foundation before they spread.

### `.help` with a `??` fallback picks the localised-string overload

[`qdvc-progress-mac` UR.3](https://github.com/qdvc-apps/qdvc-progress-mac/blob/c5e00bc3b056c1717244562044646ccdb732418a/ai-assistance-logs/2026-10-21-claude.md#3-user-request)

**Symptom:** `'appendInterpolation' is deprecated: Localized string interpolation produces an unlocalized, debug description for this type of value`.

**Cause:** The `?? "fallback"` made Swift infer a `LocalizedStringKey` for `.help(...)`, and interpolating a `Character` into one is deprecated.

**Fix:** Build the tooltip in a helper that returns a plain `String`, so the `String` overload is used.

### Combine types in a declaration warn "'Combine' was not imported by this file"

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-macos-swift-combine-not-imported-warning.md)

**Symptom:** A SwiftUI view that imported only `SwiftUI` and `AppKit` and stored a timer publisher as a property (`let ticker = Timer.publish(every: 1, on: .main, in: .common).autoconnect()`) warned: `cannot use generic class 'Autoconnect' in a property declaration member of a type not marked '@_implementationOnly'; 'Combine' was not imported by this file`, plus a similar warning about `Publishers`. Older SwiftUI examples use the same code without `import Combine`.

**Cause:** The property's inferred type, `Publishers.Autoconnect<Timer.TimerPublisher>`, comes from Combine, which the file could only see indirectly through another import, and the compiler warns when such a type appears in a declaration. The best guess is that newer SDKs (Xcode 26 or later here) no longer make Combine's types fully visible through `import SwiftUI`.

**Fix:** Add `import Combine` to every file that uses Combine types in declarations, including model files that use `ObservableObject` or `@Published` and import only `Foundation`. (Not confirmed.)

**Watch for:** Any declaration whose type comes from a framework the file doesn't import directly: `AnyCancellable` properties, publishers stored for `onReceive`. If you lint imports against an allowlist, add `Combine` to it.

### `Darwin.stat(path, &buf)` resolves to the struct, not the function

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-macos-swift-darwin-stat-resolves-to-struct.md)

**Symptom:** Inside a type that also has its own method called `stat`, a call to the C function failed: `argument passed to call that takes no arguments` on `Darwin.stat(path, &st) == 0`, followed by `referencing operator function '==' on 'BinaryInteger' requires that 'stat' conform to 'BinaryInteger'`. The neighbouring `Darwin.lstat(path, &st)` compiled fine.

**Cause:** In Darwin, `stat` is both a C struct and a C function. With the module qualifier, `Darwin.stat(…)` was resolved as the struct's no-argument initialiser, and the second error follows from comparing a struct with `0`. The qualifier was only there because, inside the type, an unqualified `stat(…)` finds the type's own method. `lstat` has no struct of the same name. (Best guess from the compiler output.)

**Fix:** Call the C function from a small private helper at file scope, outside the type, where the unqualified name resolves to the function by overload:

```swift
private func cStat(_ path: String, _ buffer: inout stat) -> Int32 {
    stat(path, &buffer)
}
```

(Not confirmed by a complete build.)

**Watch for:** Other C names that are both a struct and a function, such as `flock` and `sigaction`. Don't name your own methods after them, or call the C function through a file-scope helper.

### A local constant hides a method with the same name

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-macos-swift-local-constant-hides-method.md)

**Symptom:** `cannot call value of non-function type 'Int'` on a line like `try archive([path])`.

**Cause:** Earlier in the same method, a local constant had the same name as an instance method (`let archive = flag ? 2 : 3`). Swift's name lookup finds the innermost declaration, the local `Int`, and doesn't fall back to the method, even though only the method makes sense with call syntax. The code was written while porting from a language where the same shape is harmless. Confirmed by renaming the constant.

**Fix:** Rename the local (`let archiveIndex = …`), or call the method as `self.archive(…)`.

**Watch for:** Local names that match a method of the enclosing type, especially short verbs and nouns (`save`, `load`, `count`, `items`), and `let x = x(…)` or `guard let x = x(…)` patterns, which can also give "use of local variable before its declaration". A quick search for locals that share a name with a method called later in the same function catches them before a build. This is also one of the [errors an assistant that can't compile tends to leave behind](#code-it-couldnt-compile-run-or-see).

### Views placed with `.offset` leave their container too narrow to hover or click

Non-public QDVC project (2026-11-15) · [report](reports/processed/2026-11-15-macos-swift-offset-does-not-change-layout.md)

**Symptom:** A custom bar made of segments laid end to end stopped responding part-way along: hovering, clicking and dragging worked over the first stretch and did nothing beyond it, although the later segments were drawn correctly. Bars with a single segment worked all the way along. The data in the later segments was suspected first, and was fine.

**Cause:** The dead zone started at the width of the longest segment. Each segment was a fixed-width view in a `ZStack(alignment: .leading)`, moved into place with `.offset(x:)`. `.offset` moves where a view is drawn, not the frame layout gives it, so the `ZStack` was only as wide as its widest child, and the `.onHover`, `.onContinuousHover` and drag gesture attached to it only covered that width. (Best explanation from the code; it fits the symptom exactly.)

**Fix:** Give the container an explicit frame across the full width (`.frame(width: totalWidth, height: …, alignment: .leading)`) and a `.contentShape(Rectangle())` before the hover and gesture modifiers. Positioning the pieces with layout (an `HStack` with computed widths, or leading padding) instead of `.offset` also works. (Not confirmed.)

**Watch for:** Any custom slider, timeline, ruler or chart built by offsetting pieces in a `ZStack`. Everything attached to the container (`onHover`, `contentShape`, gestures, `.background`, `.overlay`) follows the pieces' un-offset frames, and thumbs or markers placed with `.offset` can be clipped or untouchable for the same reason. A symptom that starts at a point matching the size of one of the parts points to layout, not data.

## macOS toolchain: Xcode and SwiftPM

### Builds with only the Command Line Tools fail on `SwiftUIMacros`

[`qdvc-progress-mac` UR.5](https://github.com/qdvc-apps/qdvc-progress-mac/blob/c5e00bc3b056c1717244562044646ccdb732418a/ai-assistance-logs/2026-10-10-claude.md#5-user-request)

**Symptom:** After a macOS/Xcode update, building with the Command Line Tools failed because `SwiftUIMacros` couldn't be found.

**Cause:** From Xcode 27, SwiftUI's `@State` is a macro whose plugin ships only with full Xcode.

**Fix:** The shared QDVC `build-app.sh` block picks the newest installed Xcode when `xcode-select` points at the Command Line Tools, builds through `xcrun swift build`, and reports the compiler and SDK. For `swift run` and `swift test`, set `DEVELOPER_DIR` or use `xcode-select --switch`. Keep this block in step across the macOS apps.

### Liquid Glass depends on the SDK version recorded in the binary

[`qdvc-progress-mac` UR.9](https://github.com/qdvc-apps/qdvc-progress-mac/blob/c5e00bc3b056c1717244562044646ccdb732418a/ai-assistance-logs/2026-10-10-claude.md#9-user-request)

**Symptom:** The app kept the classic look instead of Liquid Glass, despite building with a new enough SDK.

**Cause:** `vtool -show-build` showed `sdk 14.0`, the deployment target. `swift build` writes that value into the binary's SDK field, and macOS reads that field to decide whether an app gets Liquid Glass.

**Fix:** The build script rewrites the field with `vtool` to the SDK actually used, before signing. A plain `swift run` build still gets the classic look.

### SwiftPM caches `Package.swift`, so adding a framework changes nothing

[`qdvc-articulate` UR.6](https://github.com/qdvc-apps/qdvc-articulate/blob/6d20951f8fd74b15815335fab082ff754e0e7534/ai-assistance-logs/2026-11-12-claude.md#6-user-request)

**Symptom:** After `llama.xcframework` was put into `Vendor/` and the build script detected it, the app still said it had no AI engine. The log showed `Build complete! (0.20 sec.)`.

**Cause:** SwiftPM caches its evaluation of `Package.swift`. Adding a folder doesn't count as a change to that file, so it kept the earlier "no llama.cpp" answer. Because the engine was gated on `canImport(llama)`, the missing framework silently produced a stub instead of an error.

**Fix:** The build script fingerprints the framework and cleans `.build` when it changes, checks with `otool` that the executable links the framework, and reports whether the engine is included. The engine is gated on an explicit flag set in `Package.swift`, so a framework that can't be imported fails the build loudly.

### Developer tools in `/usr/bin` can't run from a sandboxed Mac app

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-macos-toolchain-usr-bin-tools-refuse-app-sandbox.md)

**Symptom:** (Found while planning; not tested.) Running `/usr/bin/git`, `python3`, `swift`, `otool` and similar tools through `Process` from an App Sandbox app fails with `xcrun: error: cannot be used within an App Sandbox.`

**Cause:** Most developer tools in `/usr/bin` are small trampolines that call `xcrun` to find the real tool in the selected Xcode or Command Line Tools, and `xcrun` refuses to run inside the App Sandbox. According to Apple Developer Technical Support on the Apple Developer Forums, there's no way around that check. This has been reported since at least 2016.

**Fix:** Don't shell out to `/usr/bin` developer tools from a sandboxed app. Implement what's needed natively, or embed your own signed copy of the tool in the app bundle if its licence allows it. Test-only code that runs outside the sandbox (such as `swift test`) can still use them.

**Watch for:** An early "just shell out to the command-line tool" plan for a Mac app that must be sandboxed. Even unsandboxed, a Mac with no developer tools installed shows an installer prompt for `/usr/bin/git` instead of running it.

## Swift, SwiftUI and UIKit (iOS and iPadOS)

### On iPad in landscape, the camera preview moves at right angles to the camera

Non-public QDVC project (2026-11-15) · [report](reports/processed/2026-11-15-new-ios-camera-preview-rotation.md)

**Symptom:** A camera preview (a QR code scanner hosted in SwiftUI through `UIViewControllerRepresentable`) worked in portrait. On an iPad in landscape, panning the iPad moved the picture at right angles, as if the preview were turned 90°.

**Cause:** An `AVCaptureVideoPreviewLayer` shows frames in the camera sensor's own orientation unless its connection is told to rotate them, and nothing updated that as the interface rotated. The older way, setting `connection.videoOrientation` from the interface orientation, has been deprecated since iOS 17. (Best explanation from the code and Apple's API; not measured.)

**Fix:** Create an `AVCaptureDevice.RotationCoordinator(device:previewLayer:)` (iOS 17 and later), observe its `videoRotationAngleForHorizonLevelPreview` with key-value observation, and set the preview connection's `videoRotationAngle` to it after checking `isVideoRotationAngleSupported`. Apply it at setup, in `viewDidLayoutSubviews` and in `viewWillTransition(to:with:)`, and keep the layer's `frame` equal to the view's bounds on every layout. (Not confirmed.)

**Watch for:** Testing only on an iPhone in portrait hides this completely; iPad apps are never portrait-only. The same applies to any custom camera view, such as document scanning or video capture. Code samples that set `videoOrientation` predate the rotation coordinator; they still compile, with a deprecation warning.

### "Unable to type-check this expression in reasonable time" from literal arithmetic in a modifier chain

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-ios-swiftui-type-check-timeout-in-modifier-chain.md)

**Symptom:** A short SwiftUI modifier chain failed to build with `The compiler is unable to type-check this expression in reasonable time; try breaking up the expression into distinct sub-expressions`, reported against the whole statement. The chain included `.opacity(Double(max(0, (1 - progress * 2) * (1 - fraction * 4))))`, `.allowsHitTesting(isVisible && progress == 0 && offset == 0)` and `.opacity(isVisible ? 1 : 0)`, with `CGFloat` state driven by a drag gesture.

**Cause:** A modifier chain is a single expression, so the type checker solves it all at once. Every untyped literal could be any numeric type, `max` is generic, `Double(…)` has many overloads, and Swift converts implicitly between `CGFloat` and `Double`, which multiplies the combinations. (Best explanation; not measured.)

**Fix:** Work out each value first as a separate constant with an explicit type (`let fade: CGFloat = …`, `let overlayOpacity: Double = …`, `let interactive: Bool = …`), then use the constants in the chain. Plain `let`s are allowed inside a `ViewBuilder`. Confirmed: the next build got past it. Three similar expressions in other files were split up pre-emptively.

**Watch for:** Gesture-driven views, with literal arithmetic on `CGFloat` state directly inside `.opacity`, `.offset`, `.scaleEffect`, `.frame` or `.rotationEffect`. Wrapping the expression in `Double(…)` or `CGFloat(…)` makes it worse; give the intermediate value a type instead. The compiler stops at the first file that times out, so search the other view files for the same shape. See also the [macOS entry on long closure chains](#privateset-doesnt-reach-an-extension-in-another-file).

## Kotlin, Jetpack Compose and Android UI

### `using` and `sizeTransform` aren't what they look like

[`qdvc-nts-android` UR.7](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#7-user-request) → [`qdvc-nts-android` UR.9](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#9-user-request) · [`qdvc-countdowns-android` UR.7](https://github.com/qdvc-apps/qdvc-countdowns-android/blob/35f09cf8e128d3fd02357ab0bebbe693864cfb57/ai-assistance-logs/2026-08-02-claude.md#7-user-request)

**Symptom:** `Unresolved reference: using` in a shared slide-transition helper. In `qdvc-nts-android` the first fix then failed too, and the identical error came back a few days later in `qdvc-countdowns-android`.

**Cause:** `using` is declared as a member of `AnimatedContentTransitionScope` (line 314 of androidx's `AnimatedContent.kt`), not as a top-level extension, so it only resolves inside a `transitionSpec` lambda, never in a plain helper function. The obvious workaround, assigning `ContentTransform.sizeTransform` afterwards, doesn't compile either, because that property has an `internal set`.

**Fix:** Build the transform with the public constructor: `ContentTransform(enter, exit, targetContentZIndex, SizeTransform { _, _ -> snap() })`. `qdvc-countdowns-android` got it right first time only after the assistant read the androidx source instead of guessing.

### `AnimatedContent`'s default `SizeTransform` makes slides drift diagonally

[`qdvc-markdown-notebook-android` UR.29](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#29-user-request)

**Symptom:** Navigating between screens sometimes slid in from a corner instead of cleanly sideways.

**Cause:** The default `SizeTransform` animates the container's height while the horizontal slide runs, so screens of different heights appear to move diagonally.

**Fix:** Pass `SizeTransform { _, _ -> snap() }` so the size changes instantly and only the slide animates. Lateral tab-switch animations need the same guard ([`qdvc-android-app-specification` UR.1](https://github.com/qdvc-apps/qdvc-android-app-specification/blob/1d6787c780e4866d0b144a98e3942af83573c52a/vibe-coding/2026-08-02-claude.md#1-user-request)).

### The `AnimatedContent` content lambda must use its own argument

[`qdvc-checklists-android` UR.9](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-07-25-claude.md#9-user-request)

**Symptom:** Tapping a workspace showed the destination screen instantly, then animated it sliding over another copy of itself.

**Cause:** The `content` lambda ignored the animated target state it was given (`it`) and read the live `browse.mode` from the enclosing scope, so both the outgoing and incoming frames rendered the new screen.

**Fix:** Always switch on the lambda's parameter, never on outer state.

### Switching tabs throws away scroll position

[`qdvc-checklists-android` UR.21](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#21-user-request)

**Symptom:** Opening an item and coming back to the list scrolled it back to the top.

**Cause:** The `AnimatedContent` that swaps tabs discards the outgoing tab's composition, so the implicitly remembered `rememberLazyListState()` is rebuilt at zero. `rememberSaveable` doesn't help: it survives configuration changes, not leaving composition.

**Fix:** Hoist the `LazyListState` above the `AnimatedContent`, keyed per item or workspace, and prune keys to what's still open so the map can't grow over a session. Deliberately let search results, reorder lists and Settings reset to the top.

### A phantom gap above the bottom navigation bar

[`qdvc-checklists-android` UR.9](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-07-25-claude.md#9-user-request) · [`qdvc-android-app-specification` UR.1](https://github.com/qdvc-apps/qdvc-android-app-specification/blob/1d6787c780e4866d0b144a98e3942af83573c52a/vibe-coding/2026-08-02-claude.md#1-user-request)

**Symptom:** About 1 cm of blank space between page content and the bottom navigation bar. It appeared independently in both `qdvc-checklists-android` and `qdvc-nts-android`.

**Cause:** Nested `Scaffold`s each applied the bottom system-bar inset on top of the outer `Scaffold`'s bottom-bar reservation. In `qdvc-nts-android` a content `Box` also re-applied the bottom inset that the `NavigationBar` already owns.

**Fix:** Exactly one owner per inset edge. The outer `Scaffold` contributes nothing (`contentWindowInsets = WindowInsets(0, 0, 0, 0)`), each screen's `TopAppBar` owns the top through its default `windowInsets`, the bottom `NavigationBar` owns the bottom, and no inner `Scaffold` adds a bottom inset. A named helper such as `topOnlyInsets = WindowInsets.systemBars.only(WindowInsetsSides.Top)` makes the intent obvious.

### The keyboard leaves a huge gap above it, or covers the input

[`qdvc-nts-android` UR.29](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#29-user-request) → [`qdvc-nts-android` UR.35](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#35-user-request)

**Symptom:** First the keyboard covered the chat composer. Then the composer rose with the keyboard but left a gap of about 2.5 cm, which survived two more attempts.

**Cause:** `enableEdgeToEdge()` sets `decorFitsSystemWindows = false`, and together with the manifest's `adjustResize` the keyboard was counted twice. Worse, the composer sat above the app's own bottom `NavigationBar`, so `imePadding()` over-lifted by the whole bar (its content plus its inset). The intuitive fix, `WindowInsets.ime.exclude(WindowInsets.navigationBars)`, only subtracts the system inset and leaves most of the gap.

**Fix:** Set `android:windowSoftInputMode="adjustNothing"`, hide the bottom `NavigationBar` while `WindowInsets.isImeVisible`, and apply a single plain `imePadding()` to the bottom-most element. `adjustResize` is fine for apps with no input anchored to the bottom of the screen. `WindowInsets.isImeVisible` needs `@OptIn(ExperimentalLayoutApi::class)`, imported from `androidx.compose.foundation.layout` ([see below](#experimentallayoutapi-isnt-where-experimentalfoundationapi-is)).

### System bars stay black on Samsung, and the nav bar colour doesn't quite match

[`qdvc-markdown-notebook-android` UR.7](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#7-user-request) → [`qdvc-markdown-notebook-android` UR.9](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#9-user-request)

**Symptom:** The status bar and the system navigation bar were plain black instead of following the app's theme.

**Cause:** Without edge-to-edge, the system draws its own bar backgrounds. Once fixed, the system nav bar still didn't quite match the app's bottom bar, because Material 3's `NavigationBar` applies a tonal-elevation tint, so its colour isn't the raw `surface` colour.

**Fix:** Call `enableEdgeToEdge`, drive light/dark bar icons through `WindowInsetsControllerCompat` from the resolved theme, and colour-match against the bottom bar's actual container colour. That applies to apps targeting SDK 34 or lower; from SDK 35, see [turning the scrim off](#from-android-15-match-the-navigation-bar-by-turning-its-scrim-off).

### `ParagraphStyle` plus a literal newline gives double line breaks

[`qdvc-markdown-notebook-android` UR.33](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#33-user-request)

**Symptom:** An empty line after every line of content in both View and Edit, and a tiny editor with a big gap above the keyboard.

**Cause:** Each line was wrapped in its own `ParagraphStyle` (to get hanging indents) and a `\n` was still appended. A `ParagraphStyle` boundary already ends the paragraph, so every break happened twice, which also inflated the content height.

**Fix:** Don't emit both. The app ended up with two highlighter paths, with hanging indents in View only; in the editable path, styles must not change the character count, or the caret mapping breaks.

### Offsets in pixels, widths in dp

[`qdvc-markdown-notebook-android` UR.9](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#9-user-request)

**Symptom:** Swiping a row to close it revealed only a sliver of the red action strip, with the "Close" label clipped.

**Cause:** The row slid by 96 pixels, but the action strip was 96 dp wide. At roughly 2.75× density, only about 35 dp was exposed.

**Fix:** Convert dp to pixels with `LocalDensity` before using it in `translationX` or any other pixel-based API.

### `Modifier.padding` can't mix `horizontal` with `top`

[`qdvc-nts-android` UR.31](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#31-user-request)

**Symptom:** `Cannot find a parameter with this name: top`.

**Cause:** `padding` has a `(horizontal, vertical)` overload and a `(start, top, end, bottom)` overload, but none that mixes them.

**Fix:** Use `start`/`end`/`top`/`bottom`.

### Don't import `item` for a `LazyColumn`

[`qdvc-checklists-android` UR.3](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-07-25-claude.md#3-user-request)

**Symptom:** `Unresolved reference: item` on an import line.

**Cause:** `item { }` is a member of `LazyListScope`, available inside the `LazyColumn` lambda without any import. Only the `items(list)` extensions are importable.

**Fix:** Remove the import.

### `horizontalScroll` and intrinsic measurements don't mix

[`qdvc-cat-android` UR.5](https://github.com/qdvc-apps/qdvc-cat-android/blob/9a510bb3d0de8fe081d7cfd15bc07483fcafb806/vibe-coding/2026-07-25-claude.md#5-user-request)

**Symptom:** (Caught before shipping.) A line-number gutter sized with `IntrinsicSize.Min` next to horizontally scrolling code would have crashed with word-wrap off.

**Cause:** Scrollable containers don't support intrinsic measurements.

**Fix:** Only use the intrinsic pass in the word-wrap layout; with wrap off, every line is one row tall anyway.

### `MATCH_PARENT` inside a `wrap_content` parent eats the whole screen

[`qdvc-paperpod-suite` UR.9](https://github.com/qdvc-apps/qdvc-paperpod-suite/blob/0958ad36101b7e32d0f8b0bd24be0ea79fd9866d/ai-assistance-logs/2026-08-17-claude.md#9-user-request)

**Symptom:** On the e-ink tablet, the icon rail took 100% of the width, so no content was visible.

**Cause:** A flexible spacer inside the rail (an Android View layout, not Compose) was `MATCH_PARENT` wide inside a `wrap_content` rail, so the rail expanded to fill the parent and the content pane got nothing. Other rows elsewhere only looked right by the same accident.

**Fix:** Give the rail an explicit width and make every such layout explicit.

### `CONTEXT_CLICK` is the faintest haptic

[`qdvc-checklists-android` UR.29](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#29-user-request)

**Symptom:** Tap feedback was barely perceptible, even with the phone's vibration intensity at maximum.

**Cause:** `HapticFeedbackConstants.CONTEXT_CLICK` maps to the platform's lightest predefined effect (about `EFFECT_TICK`).

**Fix:** Use `VIRTUAL_KEY` for taps and keep the faint constants for repeating sensations. Use `View.performHapticFeedback`, not `Vibrator`, so the system setting is honoured and no `VIBRATE` permission is needed.

### `ExperimentalLayoutApi` isn't where `ExperimentalFoundationApi` is

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-android-ui-experimentallayoutapi-package.md)

**Symptom:** Four errors from one import line: `Unresolved reference 'ExperimentalLayoutApi'` (twice), `Annotation argument must be a compile-time constant`, and `The API of this layout is experimental and is likely to change in the future` on a use of `WindowInsets.isImeVisible`.

**Cause:** The import was `androidx.compose.foundation.ExperimentalLayoutApi`, by analogy with `androidx.compose.foundation.ExperimentalFoundationApi`. But `ExperimentalLayoutApi` lives one package deeper, in `androidx.compose.foundation.layout`, beside the layout APIs it marks. The other three errors follow from the first: with the marker unresolved, the `@OptIn` is invalid, so the experimental-API check fires anyway.

**Fix:** `import androidx.compose.foundation.layout.ExperimentalLayoutApi`. Confirmed: the next build got past all four errors.

**Watch for:** Compose's opt-in markers live beside the APIs they mark (`…foundation.ExperimentalFoundationApi`, `…foundation.layout.ExperimentalLayoutApi`, `…material3.ExperimentalMaterial3Api`). If `@OptIn` gives "Annotation argument must be a compile-time constant", the marker class itself didn't resolve: fix the import first.

### An `Int` SDK constant turns an `if` into `Number & Comparable<*>`

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-android-ui-int-sdk-constant-in-long-if-expression.md)

**Symptom:** `Argument type mismatch: actual type is 'Number & Comparable<CapturedType(*)>', but 'Long' was expected.`, on a call to `JobInfo.Builder.setEstimatedNetworkBytes`.

**Cause:** `JobInfo.NETWORK_BYTES_UNKNOWN` is declared as a Java `int`, although the methods that accept it take a `long`. Kotlin never widens `Int` to `Long` implicitly, so the branches of `if (bytes > 0) bytes else JobInfo.NETWORK_BYTES_UNKNOWN` (a `Long` and an `Int`) unify to their common supertype, which is the strange type in the message. A literal `0` would have been fine, because literals take the expected type; a named constant doesn't.

**Fix:** `JobInfo.NETWORK_BYTES_UNKNOWN.toLong()` (and `0L` for clarity). Confirmed: the next build got past it.

**Watch for:** Any Java constant declared `int` but meant for a `long` parameter, typically an "unset" or "unknown" sentinel. If an error mentions `Number & Comparable<…>`, look for an `if`, `when` or `?:` whose branches mix `Int` and `Long`.

### From Android 15, match the navigation bar by turning its scrim off

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-android-ui-nav-bar-colour-ignored-from-android-15.md)

**Symptom:** (Found while carrying an older app's approach into a new one, before running on a device.) Apps targeting SDK 34 match the system navigation bar to the app's bottom bar by setting `window.navigationBarColor` (and `statusBarColor`). For apps targeting SDK 35 or later, that no longer applies.

**Cause:** From the platform documentation and the androidx source (`EdgeToEdge.kt`); not yet measured. Android 15 forces apps targeting SDK 35+ edge to edge. The status bar and gesture navigation bar are transparent, but the three-button navigation bar gets a translucent system scrim unless `Window.setNavigationBarContrastEnforced(false)`, and `setNavigationBarColor`/`setStatusBarColor` are deprecated (and flagged by Play). `enableEdgeToEdge()`'s default navigation bar style is `SystemBarStyle.auto(…)`, which leaves contrast enforcement on; only an explicit `SystemBarStyle.light(…)` or `.dark(…)` turns it off. On API 35+ androidx also draws a non-transparent scrim as its own protection view rather than as the window colour.

**Fix:** Call `enableEdgeToEdge` with explicit, fully transparent styles chosen from the resolved theme (`SystemBarStyle.dark(Color.TRANSPARENT)`, or `SystemBarStyle.light(Color.TRANSPARENT, Color.TRANSPARENT)`), and set `window.isNavigationBarContrastEnforced = false` for clarity. Let the app's bottom `NavigationBar` supply the colour (`containerColor = MaterialTheme.colorScheme.surface`, `tonalElevation = 0.dp`, default `windowInsets`, so it extends behind the buttons). Call it again when the theme changes and after full-screen views. (Not confirmed on a device.)

**Watch for:** Gesture navigation has no scrim either way, so test with three-button navigation. Screens without the bottom bar need their own background behind the transparent navigation bar. `enableEdgeToEdge()` with no arguments isn't "transparent".

### A `var` with `private set` still clashes with your own `setX()`

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-android-ui-private-setter-still-clashes-with-setx.md)

**Symptom:** `Platform declaration clash: The following declarations have the same JVM signature (setTheme(Ljava/lang/String;)V)`, listing the property's generated setter and a hand-written `fun setTheme(value: String)`. Gradle summarised it as `CONFLICTING_JVM_DECLARATIONS`.

**Cause:** Every Kotlin `var` compiles to JVM `getX`/`setX` methods. A `private set` only changes the setter's visibility; the JVM method still exists, and the JVM doesn't allow two methods with the same name and parameter types in one class, whatever their visibility. Delegated properties (`by mutableStateOf(…)`) behave the same. The code was ported from Swift, where `private(set) var scale` beside `func setScale(_:)` is a common, legal pattern.

**Fix:** Rename the functions (`setScale` became `changeScale`). Alternatives: drop the `private set`, or rename the setter's JVM name with `@set:JvmName("…")`. (The renaming cleared the reported errors; the rest of the build wasn't confirmed.)

**Watch for:** The same clash between `val x` and `fun getX()`, and between `val isX` and `fun isX()`. It's most likely when porting from Swift. Search for `fun set[A-Z]` and `fun get[A-Z]` and check each against the class's properties, because the compiler only reports it at the JVM-signature stage.

### `by rememberUpdatedState(…)` needs `import androidx.compose.runtime.getValue`

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-android-ui-state-delegate-needs-getvalue-import.md)

**Symptom:** `Type 'State<(String) -> Unit>' has no method 'getValue(Nothing?, KProperty0<*>)', so it cannot serve as a delegate.`, followed by an unresolved reference to the delegated property.

**Cause:** Property delegation calls operator functions named `getValue` (and `setValue` for a `var`). For Compose's `State` and `MutableState` these are extension functions in `androidx.compose.runtime`, not members, so they only apply when imported. Android Studio usually adds the import when you type `by`, so this mostly bites code written outside the IDE.

**Fix:** `import androidx.compose.runtime.getValue`, plus `import androidx.compose.runtime.setValue` for any `var … by`. Confirmed: the next build got past both errors.

**Watch for:** The message looks like a type problem but is a missing import. Check every file with `by remember`, `by mutableStateOf`, `by produceState`, `by animate…AsState` or `by rememberUpdatedState`; one `getValue` import covers all the overloads.

### A Media3 `PlayerView` in Compose plays sound but no picture

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-android-ui-surfaceview-video-hidden-under-compose.md)

**Symptom:** A short video was meant to replace a still image (its poster frame) while the user held a finger on it. The sound played, but the picture never appeared and the still stayed on screen. The video was a Media3 `PlayerView` hosted through `AndroidView`, in the same `Box` as the still, inside a zoomable area using `graphicsLayer`.

**Cause:** `PlayerView` renders into a `SurfaceView` by default. A `SurfaceView`'s surface is a separate layer composited behind the app's window, shown through a transparent hole the view punches in it, so anything the window draws over that area (here, the still) hides the video. It also doesn't follow Compose's draw order, alpha, clipping or `graphicsLayer` transforms. The player was running normally; its frames just went to a surface nobody could see. (Best explanation: the fix changed two things at once.)

**Fix:** Render into a `TextureView` instead (`factory = { TextureView(it).also { player.setVideoTextureView(it) } }`, with `player.clearVideoSurface()` on dispose), placed in front of the still and sized to the video's aspect ratio from `onVideoSizeChanged`. Moving the video in front of the still may have been enough on its own. `PlayerView` also accepts `app:surface_type="texture_view"`, but only from XML, and newer Media3 releases have a `PlayerSurface` composable in `media3-ui-compose` with a texture-view option. (Not confirmed.)

**Watch for:** Any `SurfaceView` hosted in Compose with content over it, or inside a pager, zoom container or animated transition: video players, camera previews (CameraX `PreviewView`'s default performance mode uses one), maps. Sound with no picture, or a picture that stays put while the screen animates, points to layering rather than decoding. For a plain full-screen video with nothing over it, `SurfaceView` is still the better choice, since `TextureView` costs a little more power and latency.

### Compose `Text` outside a `Surface` is black, even in dark mode

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-android-ui-text-outside-surface-is-black.md)

**Symptom:** In dark mode, some lines in a full-screen overlay's details panel were black on the dark background and almost invisible; others in the same panel were fine. The black lines were plain `Text(…)` calls with no colour. Light mode looked correct.

**Cause:** Material 3's `Text` takes its colour from `color`, then the style's colour, then `LocalContentColor`, which defaults to `Color.Black` outside any provider. `Surface` provides it (and `Scaffold`, `TopAppBar`, `NavigationBar` and cards all draw a `Surface`); `MaterialTheme` itself doesn't. The overlay was a plain `Box` drawn above the `Scaffold`, so it was outside every `Surface`. (From the Material 3 API; fits the symptom exactly.)

**Fix:** Wrap the content in `CompositionLocalProvider(LocalContentColor provides MaterialTheme.colorScheme.onSurface)`, and add the same provider once at the app's root as a safety net (a `Surface` inside still overrides it). (Not reported again, but not explicitly confirmed.)

**Watch for:** Overlays, full-screen viewers and `AnimatedVisibility` layers beside or over a `Scaffold`, and screens built from plain `Box`es and `Column`s. Untinted `Icon`s use `LocalContentColor` too. Light mode hides the bug, so check every screen in dark mode.

### A `Unit` suspend call used as a value crashes with `Float cannot be cast to kotlin.Unit`

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-android-ui-unit-suspend-call-leaks-float-classcastexception.md)

**Symptom:** Tapping the current bottom-navigation tab to scroll back to the top crashed on one tab only: `java.lang.ClassCastException: java.lang.Float cannot be cast to kotlin.Unit`, in the launch lambda's `invokeSuspend`. The code was `scope.launch { when (tab) { … -> gridState.animateScrollToItem(0); Tab.C -> scrollState.animateScrollTo(0) } }`, and only the `ScrollState` branch (a `verticalScroll` column, not a lazy grid) crashed.

**Cause:** Best explanation from the stack trace and how Kotlin compiles suspend calls; not reproduced in isolation. `ScrollState.animateScrollTo` is declared to return `Unit`, but its body is understood to end in a call that returns a `Float` (the amount scrolled). A suspend call in tail position passes the caller's continuation straight on, so when it really suspends (an animation always does), the caller is resumed with that `Float`. Usually nobody looks at the result. Here the `when` was a `Unit` expression and the last expression of the lambda, so the compiler cast the resumed value to `Unit`, and the cast failed. (A first explanation, that the branches' types were merged, was wrong: the declared type is `Unit`.)

**Fix:** Call the scroll functions as statements in their own `suspend fun` that returns `Unit` explicitly, and launch that. Ending the lambda with an explicit `Unit` after the `when` should also work. (Not confirmed.)

**Watch for:** A `Unit` suspend call whose result is used as a value: the last expression of a lambda inside `if` or `when`, `val x = …`, after `?:`, or through a generic `T` (`async`/`await`, `runCatching`, `withContext`). It only crashes when the call actually suspends, so instant paths and tests with an immediate dispatcher pass. `X cannot be cast to kotlin.Unit` in `invokeSuspend` is the tell.

## Android storage and data: SAF and Room

### After `renameDocument`, `fromSingleUri` silently can't write

[`qdvc-nts-android` UR.19](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#19-user-request)

**Symptom:** Backdating a note renamed its folder but left its `README.md` unchanged, and the note could no longer be opened.

**Cause:** `DocumentsContract.renameDocument` returns a new document URI. Wrapping it with `DocumentFile.fromSingleUri` gives a single-document `DocumentFile`, on which `findFile` and `createFile` silently return `null`. Renaming also allocates a new document ID, so anything holding the old ID is stale.

**Fix:** Re-resolve the renamed folder through the granted tree (by document ID, with a name-based fallback) so it's a tree-backed `DocumentFile`. Match items by a stable app-level ID, not the document ID.

### Browsing into a subfolder shows it as empty

[`qdvc-markdown-notebook-android` UR.5](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#5-user-request)

**Symptom:** Subfolders with notes appeared empty in the app.

**Cause:** The code manufactured a new tree URI for the child folder instead of navigating within the granted tree, and listing children that way didn't work.

**Fix:** Keep the original workspace tree URI as the permission root and navigate children by document ID within that tree, using `buildChildDocumentsUriUsingTree`.

### SAF document IDs leak folder paths

[`qdvc-checklists-android` UR.15](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-07-25-claude.md#15-user-request)

**Symptom:** The app's CSV logs exposed the workspace's location and name.

**Cause:** SAF document IDs embed the path (for example `primary:MyFolder/BCL091-…/01-…`).

**Fix:** Never store or export document IDs. Use workspace-relative identifiers such as folder names.

### Appending with `"wa"` can fuse two CSV rows into one

[`qdvc-checklists-android` UR.17](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#17-user-request)

**Symptom:** A malformed log row appeared, made of two valid rows spliced together.

**Cause:** `openOutputStream(uri, "wa")` writes straight onto the final byte. If the last line lacked a terminator, the new row joined it and both were destroyed. A related latent bug: a read helper returning an empty list for both "empty file" and "read failed" made the rewrite fallback truncate the file.

**Fix:** Classify the final byte (accept both CR and LF) and write the missing separator first. Find the tail by streaming to the end rather than trusting the directory listing's size, which is stale exactly when another client has just written. If the tail can't be determined, fall back to a full rewrite. Make failed reads distinguishable from empty ones.

### A synchronous Room query on the main thread crashes the app

[`qdvc-checklists-android` UR.27](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-07-25-claude.md#27-user-request)

**Symptom:** The app crashed on launch. Deleting the `logs` folder looked like the trigger but was a red herring.

**Cause:** Opening a workspace called `IndexRepository.statusFor()` directly on the main thread, and Room throws `IllegalStateException` for main-thread queries unless `allowMainThreadQueries` is set.

**Fix:** Run every DAO call off the main thread. (Separately, reads stopped creating the `logs` folder; only the write path creates it.)

### Room's KSP generates Java, so Java keywords break the build

[`qdvc-checklists-android` UR.13](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#13-user-request)

**Symptom:** `IndexDao_Impl.java:605: error: <identifier> expected`, in generated code.

**Cause:** A DAO parameter was named `new`, which is valid Kotlin but a reserved word in Java, so the generated Java didn't compile.

**Fix:** Rename the parameter. Check every DAO parameter, entity property and projection field against the Java keyword list.

### Room supports FTS4, not FTS5

[`qdvc-markdown-notebook-android` UR.37](https://github.com/qdvc-apps/qdvc-markdown-notebook-android/blob/5de7ef05a76303c86cc62d4818f4ee61509d0404/vibe-coding/2026-07-18-claude.md#37-user-request)

**Symptom:** (Design constraint.) Full-text search was wanted.

**Cause:** Room only exposes FTS3/FTS4, because FTS5 isn't available in the SQLite shipped with older Android versions.

**Fix:** Use `@Fts4`. FTS5 would mean giving up Room's schema management and hand-writing virtual-table SQL and triggers.

## Android platform: permissions, networking and background work

### Targeting Android 17, local network connections just time out

Non-public QDVC project (2026-11-19) · [report](reports/processed/2026-11-19-new-android-17-local-network-permission.md)

**Symptom:** (Found while raising targetSdk to 37, before running on a device.) An app that talks to a server on the home network by IP address or `.local` name would see its connections time out, which it would take to mean "server unreachable", with no error naming a permission.

**Cause:** From the [Android documentation](https://developer.android.com/privacy-and-security/local-network-permission); not yet measured. For apps targeting Android 17 (SDK 37) or higher, local network traffic is blocked unless the user grants `ACCESS_LOCAL_NETWORK`. Apps targeting 36 or lower keep an implicit grant through `INTERNET`, but only until they raise targetSdk. It's a runtime permission in the `NEARBY_DEVICES` group (users who already granted, say, Bluetooth aren't asked again). Blocked TCP connections typically time out and UDP gets `EPERM`. It applies to every networking API (sockets, `HttpURLConnection`, OkHttp, `NsdManager`) and to resolving `.local` names; internet traffic, mobile data and DNS to a local server on port 53 are unaffected.

**Fix:** Declare `<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />`, request it with a short explanation before the first connection, and check it before connecting, but only when `Build.VERSION.SDK_INT >= 37` (on earlier versions the permission doesn't exist and would always read as denied). Show a "not allowed" state rather than "unreachable", offer the app's settings page if the system won't ask again, and re-check in `onResume`. Where it fits, the system pickers (such as `NsdManager`'s mDNS picker) avoid the broad permission altogether. (Not confirmed on a device.)

**Watch for:** Background work needs the permission too, since the grant belongs to the app. Any app that talks to a LAN server, NAS, printer, speaker or local web UI is affected the day it targets 37, even if none of its code changed. On Android 16 the restriction can be tried early with `adb shell am compat enable RESTRICT_LOCAL_NETWORK <package>`.

## Android build tooling: Gradle and Kotlin

### The Compose compiler plugin needs Kotlin 2.0

[`qdvc-csv-android` UR.3](https://github.com/qdvc-apps/qdvc-csv-android/blob/24ffde8716485f237154fa20f34273cb465217ff/vibe-coding/2026-07-25-claude.md#3-user-request)

**Symptom:** The build failed at line 1 of `build.gradle.kts`.

**Cause:** The `org.jetbrains.kotlin.plugin.compose` plugin only exists from Kotlin 2.0.0. With Kotlin 1.9.x, Compose is configured through `composeOptions { kotlinCompilerExtensionVersion = … }`.

**Fix:** Upgrade to Kotlin 2.0.21 and drop `kotlinCompilerExtensionVersion`. Note that, as of August 2026, the other QDVC Android apps were still on Kotlin 1.9.x with extension version `1.5.14` and Compose BOM `2024.06.00`, so match whichever path the sibling app you're copying uses ([`qdvc-android-app-specification` UR.1](https://github.com/qdvc-apps/qdvc-android-app-specification/blob/1d6787c780e4866d0b144a98e3942af83573c52a/vibe-coding/2026-08-02-claude.md#1-user-request)).

## Windows, .NET and WinForms

### WFO1000: public settable properties on controls fail the build

[`qdvc-nice-mail-windows` UR.11](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#11-user-request)

**Symptom:** `error WFO1000: Property 'FallbackEmojiFont' does not configure the code serialization for its property content`.

**Cause:** Newer WinForms analyzers require any public settable property on a control to say whether the form designer should serialise its value into generated code.

**Fix:** If nothing outside the class sets it, make it private or read-only (what was done here). Otherwise add `[DesignerSerializationVisibility(DesignerSerializationVisibility.Hidden)]` or a `[DefaultValue]`.

### `TabControl` tabs are barely visible on Windows 11

[`qdvc-nice-mail-windows` UR.13](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#13-user-request)

**Symptom:** Very poor tab contrast in both light and dark mode.

**Cause:** With visual styles on, `TabControl` hands its drawing to the Windows theme engine, whose tab design is pale and largely unchanged since Windows 7. The dark-mode support added in .NET 9 and 10 handles tab headers poorly.

**Fix:** Owner-drawing the tab headers (`TabDrawMode.OwnerDrawFixed`) only controls the tab rectangles, not the strip and border. The chosen fix was to replace the tab strip with a `ToolStrip` of checkable buttons and a custom renderer, with a panel showing the matching page.

### "No .NET SDKs were found" with the SDK installed

[`qdvc-nice-mail-windows` UR.17](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#17-user-request)

**Symptom:** The SDK installed through Scoop wasn't picked up.

**Cause:** `dotnet` looks for SDKs next to whichever `dotnet.exe` runs. A runtime-only copy in `C:\Program Files\dotnet`, added by some installer, was on the system PATH, which Windows searches before the user PATH.

**Fix:** Remove `C:\Program Files\dotnet` from the system PATH (apps find runtimes through the registry), or install the SDK there too with `winget install Microsoft.DotNet.SDK.10`. A later runtime install may add the PATH entry back.

### WinForms `ListView` draws emoji in black and white

[`qdvc-nice-mail-windows` UR.1](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#1-user-request) → [`qdvc-nice-mail-windows` UR.5](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#5-user-request)

**Symptom:** The emoji picker's list showed monochrome emoji (the copied characters are still correct and appear in colour where pasted).

**Cause:** Classic GDI-based controls draw Segoe UI Emoji without colour.

**Fix:** Accepted as a known limitation. The colour option is to render each glyph through DirectWrite/Direct2D with colour fonts enabled and cache it as a bitmap, which needs a wrapper such as Vortice.Windows or CsWin32, since .NET doesn't include those APIs.

## Web: HTML, CSS, SVG, JavaScript and Plotly

### Plotly Sankey labels look "ghosted"

[`qdvc-gtd-eml` UR.11](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-07-03-claude.md#11-user-request) → [`qdvc-gtd-eml` UR.17](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-07-03-claude.md#17-user-request)

**Symptom:** In Plotly 3.7.0 on Chromium, Sankey node labels looked as though they were drawn several times over.

**Cause:** Plotly applies an inline `style` on each label's `<text>` element with a four-way `text-shadow` halo (four dark offset copies, which read as ghosting). Two earlier theories were wrong: `responsive: true` accumulating text nodes, and draggable nodes maintaining a second layer (making the Sankey `arrangement: "fixed"` removed dragging and fixed nothing). A CSS rule aimed at `.text-shadow` didn't match because in this version the element's class is `node-label`.

**Fix:** `text.node-label { text-shadow: none !important; }`. `!important` is what lets a stylesheet rule beat an inline style. Confirmed fixed in [`qdvc-gtd-eml` UR.19](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-07-03-claude.md#19-user-request).

**Watch for:** Ask for the browser inspector's rendered element early. It settled in one round what three rounds of theory hadn't.

### Inlined JSON can close your `<script>` tag

[`qdvc-gtd-eml` UR.5](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-08-05-claude.md#5-user-request)

**Symptom:** (Caught during hardening.) An email body containing `</script>` ended up verbatim in the generated `mail.js`.

**Cause:** It was harmless in an external script file, but if that data were ever inlined into the HTML page, the browser would end the script element there, and email content is untrusted.

**Fix:** When serialising JSON for JavaScript, escape `<`, `>` and `&` as `\uXXXX`, as well as U+2028 and U+2029. The JSON round-trips identically.

### `table-layout: fixed` ignores widths on body cells

[`qdvc-quickprisma` UR.9](https://github.com/qdvc-apps/qdvc-quickprisma/blob/5f1b00d35ef9eaab1ca2dc3b17f8ebc8773716ef/ai-assistance-logs/2026-08-12-claude.md#9-user-request)

**Symptom:** The narrow `#` column took a full third of the table.

**Cause:** With `table-layout: fixed`, column widths come from the first row or a `<colgroup>`. The header cells had no widths, so the width on the body `<td>` was ignored and the columns split evenly.

**Fix:** Declare widths in a `<colgroup>`.

### Buttons glow white in dark mode

[`qdvc-health-dashboard` UR.37](https://github.com/qdvc-apps/qdvc-health-dashboard/blob/d7fc5141a495ab0503ac9eb55e623d789cde8f04/vibe-coding/2026-07-04-claude.md#37-user-request)

**Symptom:** Inactive tabs (styled `<button>`s) glowed white on the dark tab track. Light mode looked fine.

**Cause:** `<button>` has a default light background, which had never been reset.

**Fix:** Set `background: transparent` on the base tab style.

### SVG gradients are per-element by default

[`qdvc-health-dashboard` UR.9](https://github.com/qdvc-apps/qdvc-health-dashboard/blob/d7fc5141a495ab0503ac9eb55e623d789cde8f04/vibe-coding/2026-07-04-claude.md#9-user-request)

**Symptom:** In a sleep chart with a clock-time gradient, the same time of night showed different colours on different bars.

**Cause:** The default `gradientUnits="objectBoundingBox"` stretches the gradient over each bar's own box.

**Fix:** Use `gradientUnits="userSpaceOnUse"` with coordinates on the chart's y-axis, so a given time maps to one colour on every bar.

## Other libraries and services

### llama.cpp: a doubled BOS, and warnings that are fine to ignore

[`qdvc-offgrid-buddies` UR.27](https://github.com/qdvc-apps/qdvc-offgrid-buddies/blob/f85e3e55e5a22c0ad1e56f78b440ab88d4fa7bbc/ai-assistance-logs/2026-09-09-claude.md#27-user-request)

**Symptom:** Three warnings when loading Gemma 4 through `llama-cpp-python`: "using full-size SWA cache", "V embeddings have different sizes … padding V cache to 1024", and a duplicate leading BOS.

**Cause:** The first two are informational. The full-size sliding-window cache is what makes saving and reusing a prompt prefix reliable. The V-cache padding happens because Flash Attention is off, and Flash Attention with SWA checkpoints was reported to crash. The duplicate BOS was a real bug: the prompt template hard-coded `<bos>`, and `create_completion` adds one by default.

**Fix:** Remove the hard-coded `<bos>` and let llama.cpp add it once. Leave the other two alone.

### A saved Gemma prompt prefix must not end mid-turn

[`qdvc-offgrid-buddies` UR.45](https://github.com/qdvc-apps/qdvc-offgrid-buddies/blob/f85e3e55e5a22c0ad1e56f78b440ab88d4fa7bbc/ai-assistance-logs/2026-09-09-claude.md#45-user-request)

**Symptom:** The buddy's first reply began with the literal text `<start_of_turn>model`.

**Cause:** The saved persona prefix ended with `<start_of_turn>model\n` (ready for the model to speak), and the first user turn was then appended after it, leaving a malformed transcript in which the model generated the control tokens as text.

**Fix:** End the saved prefix at `<end_of_turn>` so appended turns stay well formed, and sanitise control tokens out of displayed output as a safeguard. Existing save-points had to be rebuilt.

### Replies stop mid-sentence at `max_tokens`

[`qdvc-offgrid-buddies` UR.47](https://github.com/qdvc-apps/qdvc-offgrid-buddies/blob/f85e3e55e5a22c0ad1e56f78b440ab88d4fa7bbc/ai-assistance-logs/2026-09-09-claude.md#47-user-request)

**Symptom:** A long reply was suddenly cut off.

**Cause:** The generation call capped replies at 512 tokens. (Stop strings were the other suspect.)

**Fix:** Raise the cap, and show a warning when generation stopped because of the length limit, rather than ending silently. Sending "continue" resumes, since the cut-off text is already in the context.

### llama.cpp renamed parameters between builds

[`qdvc-articulate` UR.8](https://github.com/qdvc-apps/qdvc-articulate/blob/6d20951f8fd74b15815335fab082ff754e0e7534/ai-assistance-logs/2026-11-12-claude.md#8-user-request)

**Symptom:** `value of type 'llama_model_params' has no member 'use_mmap'`.

**Cause:** llama.cpp build b10875 (9 September 2026) replaced `use_mmap`/`use_mlock` with a load mode. The default still memory-maps the model.

**Fix:** Drop the line. Expect other renames when building against a newer llama.cpp than the code was written for, and collect them all at once with `build.sh 2>&1 | grep -E "error:" | sort -u`.

### Textual: early screen titles, and bubbles that collapse

[`qdvc-offgrid-buddies` UR.35](https://github.com/qdvc-apps/qdvc-offgrid-buddies/blob/f85e3e55e5a22c0ad1e56f78b440ab88d4fa7bbc/ai-assistance-logs/2026-09-09-claude.md#35-user-request) · [`qdvc-offgrid-buddies` UR.45](https://github.com/qdvc-apps/qdvc-offgrid-buddies/blob/f85e3e55e5a22c0ad1e56f78b440ab88d4fa7bbc/ai-assistance-logs/2026-09-09-claude.md#45-user-request)

**Symptom:** A crash when opening a chat screen; later, short and long chat bubbles all collapsed to a narrow ribbon, leaving an empty "valley" down the middle.

**Cause:** Setting `self.title` on a `Screen` updates the `Header`, which races if the header isn't mounted yet. The bubbles had `shrink=True` plus `min-width: 32`, and their `max-width` resolved against a parent with no definite width.

**Fix:** Give screens a `TITLE` class attribute (or set the title after refresh). Remove `shrink` and `min-width`, and give the bubble's parent a definite width.

**Watch for:** Rendering Markdown while streaming tokens re-parses the whole widget on every update. It caused enough trouble here that it was removed.

### WeasyPrint's flexbox doesn't constrain heights like a browser

[`qdvc-family-noticeboard` UR.13](https://github.com/qdvc-apps/qdvc-family-noticeboard/blob/8a6c46a54f062a38c549d4e35335831228774dbf/vibe-coding/2026-07-22-claude.md#13-user-request) → [`qdvc-family-noticeboard` UR.17](https://github.com/qdvc-apps/qdvc-family-noticeboard/blob/8a6c46a54f062a38c549d4e35335831228774dbf/vibe-coding/2026-07-22-claude.md#17-user-request)

**Symptom:** A panel's border bled past the page margin, even when both panels were emptied of content.

**Cause:** Never fully isolated. `height: 100%` and `flex: 1` inside a fixed-height flex column didn't bound the box the way browsers do, and a body at `height: 100%` didn't reliably equal one page.

**Fix:** The design was simplified: panels lost their full borders and fixed heights, and only need to be as tall as their content. Don't build page-filling flex layouts for WeasyPrint without testing them first.

### matplotlib: measure text in the same units as the box

[`qdvc-gioia-diagram` UR.3](https://github.com/qdvc-apps/qdvc-gioia-diagram/blob/93015ef3b04b64fc535d373653fec89f36e2a06e/vibe-coding/2026-07-20-claude.md#3-user-request)

**Symptom:** Text bled out of diagram boxes.

**Cause:** Text was measured in points but the width budget was computed in pixels, before the final figure size was set.

**Fix:** Convert axis units to points correctly, and hard-break any single word too wide for the box.

### Werkzeug over TLS logs a client hanging up as `ssl.SSLError`

Non-public QDVC project (2026-11-15) · [report](reports/processed/2026-11-15-other-libs-werkzeug-ssl-error-on-hang-up.md)

**Symptom:** A Flask app served over HTTPS by Werkzeug's own server (`run_simple` with an SSL context) logged "Error on request" tracebacks ending in `ssl.SSLError: [SYS] unknown error (_ssl.c:2501)` while sending large files. It looked like a server crash and was a red herring: the real problem was elsewhere.

**Cause:** Confirmed by reproducing it with a real TLS server. Browsers routinely abandon a large response part-way, for example to request a different byte range when seeking. Over plain HTTP that raises a `ConnectionError`, which Werkzeug treats as a dropped connection and doesn't log. Over TLS the same hang-up can surface as a bare `ssl.SSLError` ("[SYS] unknown error", "bad length" and others, depending on Python and OpenSSL versions), which is an `OSError` but not a `ConnectionError`, so Werkzeug logs a full traceback. Werkzeug then drains the request body, which raises a second SSL error ("unexpected eof while reading").

**Fix:** A `WSGIRequestHandler` subclass that wraps both `rfile` and `wfile` and turns any `ssl.SSLError` into a `ConnectionResetError`, passed to `run_simple(..., request_handler=…)`. Both directions must be wrapped, because of the drain after a failed write. The [report](reports/processed/2026-11-15-other-libs-werkzeug-ssl-error-on-hang-up.md) has the code. Confirmed in a local reproduction (no tracebacks for the same four hang-ups, with range requests and form posts still working); not confirmed on the original server.

**Watch for:** On a TLS server, this traceback usually just means a client went away, so look for whatever else went wrong. Werkzeug's development server isn't meant for production use.

### Immich's metadata search returns the hidden video half of each Live Photo

Non-public QDVC project (2026-11) · [report](reports/processed/2026-11-01-other-libs-immich-search-returns-live-photo-motion-halves.md)

**Symptom:** Listing an album with `POST /search/metadata` (`{"albumIds": […], "withExif": true, …}`) returned two items for every iPhone Live Photo: the HEIC still and a `.mov` video with no thumbnail. Every request for those videos' thumbnails failed with HTTP 404. The Immich web app shows one item per Live Photo.

**Cause:** Immich keeps the moving half of a Live Photo (or Android motion photo) as a separate `VIDEO` asset, linked from the still's `livePhotoVideoId` and marked `visibility: "hidden"`. As observed on server v3.3, the metadata search filtered by `albumIds` returns these hidden assets too, though Immich's own timelines and album views leave them out, and the server doesn't make thumbnails for them. (Confirmed by the failing items; the 404 explanation is from the Immich source.) A first theory, missing thumbnails for photos edited in Immich, was a dead end.

**Fix:** Filter listings on the client: drop assets whose `visibility` is `"hidden"` (or `"locked"`), and any asset whose `id` is another asset's `livePhotoVideoId` (in case an older server omits `visibility`). Fetch the motion half through the still's `livePhotoVideoId` and treat it as part of the still. Clean up any stray videos already stored. Confirmed: the failures stopped. Asking the server for `"visibility": "timeline"` is an alternative, but may also drop archived assets in the album (not verified).

**Watch for:** Any Immich endpoint used to list assets. Check `visibility` and `livePhotoVideoId` before counting, downloading or displaying, or album counts and storage totals will be inflated. Repeated 404s on `/assets/{id}/thumbnail` for `.mov` items are the tell.

### networkx layouts need numpy, which a plain `pip install networkx` doesn't install

Non-public QDVC project (2026-11-25) · [report](reports/processed/2026-11-25-other-libs-networkx-layout-needs-numpy.md)

**Symptom:** A web page that draws a small graph returned HTTP 500 on the server with `ModuleNotFoundError: No module named 'numpy'`. It worked in development, the tests passed, and `import networkx` worked on the server; only the call that laid out the graph failed.

**Cause:** networkx lists numpy, scipy and matplotlib only under its optional `default` extra (`importlib.metadata.requires("networkx")` shows `numpy>=2.2; extra == "default"`), so a plain `networkx` in `requirements.txt` installs none of them. Most of networkx is pure Python, and the functions that need numpy import it only when called, so the error appears at the first call. In networkx 3.7, with numpy and scipy hidden, `spring_layout`, `circular_layout`, `kamada_kawai_layout` and `pagerank` failed, while `shortest_path`, `connected_components`, `density`, `betweenness_centrality` and `eigenvector_centrality` still worked. The development environment already had numpy, which hid the gap. (Confirmed from the package metadata and by reproducing it with `sys.modules["numpy"] = None`.)

**Fix:** Either depend on `networkx[default]` (or list `numpy` explicitly), or avoid the functions that need it. Here the graphs were small, so a short, seeded pure-Python Fruchterman–Reingold layout replaced `spring_layout`. A regression test runs the graph pages in a fresh interpreter with numpy, scipy, pandas and matplotlib hidden. (Works in tests; not yet re-checked on the server.)

**Watch for:** Any library that keeps heavy dependencies in an extra: read its metadata, not just its import. Which networkx functions need numpy can change between releases. Development machines and AI sandboxes with a preinstalled scientific stack hide missing dependencies, so build a fresh virtual environment from the requirements file before release, or add a test that hides modules the project doesn't declare.

## File formats and standards

### HTML to Markdown: whitespace at emphasis edges matters

[`qdvc-html-to-markdown-SPA` UR.3](https://github.com/qdvc-apps/qdvc-html-to-markdown-SPA/blob/980445095be82b98236231c028c85426610f6f6f/vibe-coding/2026-07-24-claude.md#3-user-request) · [`qdvc-html-to-markdown-SPA` UR.7](https://github.com/qdvc-apps/qdvc-html-to-markdown-SPA/blob/980445095be82b98236231c028c85426610f6f6f/vibe-coding/2026-07-24-claude.md#7-user-request)

**Symptom:** Output like `*Calendly → email with*[*callback scam*]*text*`, and punctuation escaped everywhere (`1\.`, `\-`, `\(`).

**Cause:** Each emphasis tag ran `.trim()` on its contents, destroying the significant space in `<i>…with </i>`, so adjacent runs collided. Emphasis markers must hug non-space content, so the whitespace has to move outside them, not disappear. Separately, every special character was escaped regardless of context, which hurt the readability of the Markdown source. Nested emphasis is also ambiguous and needs a deliberate rule.

**Fix:** Move leading and trailing whitespace outside the markers. Escape only where a character is special in context (a period after a number at the start of a line, a dash at the start of a line, brackets in link syntax).

### Email: headers, bodies and encodings

[`qdvc-gtd-eml` UR.19](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-06-23-conversation-with-claude.md#19-user-request) · [`qdvc-nice-mail-windows` UR.3](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#3-user-request)

**Symptom:** (Several near misses.) Searching message bodies for a reference, and generating `.eml` files in three ports.

**Cause:** Bodies are often base64 or quoted-printable, so they must be decoded before searching. Non-ASCII header text must be RFC 2047 encoded words, and different libraries may split encoded words differently while decoding to the same text. RFC 5322 limits line length, so long body lines need base64 (or quoted-printable).

**Fix:** Decode bodies before searching, encode headers properly, switch transfer encoding when any line is too long, and compare ports on decoded content rather than raw bytes.

### Markdown piped to `glow` interprets the email body

[`qdvc-gtd-eml` UR.17](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-06-23-conversation-with-claude.md#17-user-request)

**Symptom:** (Flagged.) A preview tool's output rendered nicely in `glow`, but bodies containing lines starting with `#`, `*` or `-` would be formatted as Markdown.

**Fix:** Put a blank line after headings, wrap header blocks in a code fence, and accept or escape the body.

### CSV for spreadsheets needs a byte-order mark

[`qdvc-autolitreview-tester` UR.3](https://github.com/qdvc-apps/qdvc-autolitreview-tester/blob/a0666d217975a1e5a2bc97e2c6f35b77e3815984/ai-assistance-logs/2026-10-02-claude.md#3-user-request)

**Symptom:** (Pre-empted.) Accented characters and curly quotes are misread when Excel or Numbers open UTF-8 CSV.

**Fix:** Write UTF-8 with a BOM.

### BibTeX from tools isn't always valid BibTeX

[`qdvc-autolitreview-tester` UR.3](https://github.com/qdvc-apps/qdvc-autolitreview-tester/blob/a0666d217975a1e5a2bc97e2c6f35b77e3815984/ai-assistance-logs/2026-10-02-claude.md#3-user-request) · [`qdvc-autolitreview-tester` UR.7](https://github.com/qdvc-apps/qdvc-autolitreview-tester/blob/a0666d217975a1e5a2bc97e2c6f35b77e3815984/ai-assistance-logs/2026-10-02-claude.md#7-user-request)

**Symptom:** Reference counts were off, and the tool under test produced citation keys containing spaces (`Smith 2020`).

**Cause:** `@string`, `@preamble` and `@comment` aren't entries, entries can use `@type(…)` as well as `@type{…}`, and `@` can appear inside field values.

**Fix:** Count only top-level entries, handle both delimiters and nested braces, tolerate spaces inside keys when verifying (trimming the ends), and report clashing keys with line numbers. For the line-tagged `refer`/EndNote format, see [parser rules inferred from typical samples](#parser-rules-inferred-from-typical-samples-instead-of-the-formats-definition).

## Mistakes the AI kept repeating

These aren't platform bugs, but they cost real rounds in more than one project and are worth guarding against in the opening prompt of a new session.

### Brace expansion and process substitution in `sh`

[`qdvc-gtd-eml-mac` UR.11](https://github.com/qdvc-apps/qdvc-gtd-eml-mac/blob/8f210e1e48dbcb84d4929f3ebe876fefde3f4064/ai-assistance-logs/2026-09-30-claude.md#11-user-request) · [`qdvc-nice-mail-windows` UR.5](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#5-user-request) · [`qdvc-nice-mail-mac` UR.3](https://github.com/qdvc-apps/qdvc-nice-mail-mac/blob/8218ffc10b9c2d5e4f4f8260cecf155a2e105547/ai-assistance-logs/2026-09-30-claude.md#3-user-request) · [`qdvc-progress-gtk4` UR.1](https://github.com/qdvc-apps/qdvc-progress-gtk4/blob/52abc25fb53a95a259e7d0c0e719d559053bdee6/ai-assistance-logs/2026-10-14-claude.md#1-user-request)

**Symptom:** A stray folder literally named `{docs,tools,scripts,Resources,sample-workspace,Sources` (and deeper variants such as `{Model,UI,Resources},tests`) committed alongside the real project tree.

**Cause:** The assistant scaffolded folders with `mkdir -p x/{a,b,c}` in a sandbox whose default shell is plain `sh`, which has no brace expansion. The same shell also lacks `<(…)` process substitution, which broke diff checks in [`qdvc-health-dashboard` UR.27](https://github.com/qdvc-apps/qdvc-health-dashboard/blob/d7fc5141a495ab0503ac9eb55e623d789cde8f04/vibe-coding/2026-07-04-claude.md#27-user-request) and [`qdvc-nts-android` UR.27](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#27-user-request).

**Fix:** Ask for `bash` explicitly, or create folders one by one. Check the ZIP's top-level listing before committing.

### Guessing an API instead of reading its source

[`qdvc-nts-android` UR.7](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#7-user-request) · [`qdvc-nts-android` UR.31](https://github.com/qdvc-apps/qdvc-nts-android/blob/a96294aa76a3d3cca04cc491e7bee0732325ed1b/ai-assistance-logs/2026-07-29-claude.md#31-user-request) · [`qdvc-checklists-android` UR.3](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-07-25-claude.md#3-user-request) · [`qdvc-countdowns-android` UR.7](https://github.com/qdvc-apps/qdvc-countdowns-android/blob/35f09cf8e128d3fd02357ab0bebbe693864cfb57/ai-assistance-logs/2026-08-02-claude.md#7-user-request)

**Symptom:** Compose code that doesn't compile: `using` outside its scope, assigning an `internal set` property, a non-existent `padding` overload, an unnecessary `item` import. The same `using` error turned up in two apps.

**Cause:** The assistant wrote from memory against a library version it couldn't compile against, and the first "fix" was often another guess.

**Fix:** Paste the exact error (Pattern Z2), and ask the assistant to read the library source for the pinned version before fixing. That's what finally fixed the `using` error in `qdvc-countdowns-android`. The same goes for file formats: read the specification, not just typical samples ([see below](#parser-rules-inferred-from-typical-samples-instead-of-the-formats-definition)).

### Code it couldn't compile, run or see

[`qdvc-empirics` UR.4](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#4-user-request) · [`qdvc-response-letter-mac` UR.5](https://github.com/qdvc-apps/qdvc-response-letter-mac/blob/dc09663fa6bc23903c3fcba82379c1e2f7d49655/ai-assistance-logs/2026-10-12-claude.md#5-user-request) · [`qdvc-checklists-android` UR.13](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#13-user-request)

**Symptom:** Nearly every Swift, Android and .NET build needed at least one round of build errors, and most GTK layout issues needed several rounds of "please check on your machine".

**Cause:** The sandbox has no Xcode, Android SDK, .NET SDK or GTK display. Only pure logic (core libraries, parsers) could be tested there.

**Fix:** Push logic out of UI files into testable core modules (as the Swift apps' cores and `qdvc-checklists-android`'s `util` did). Collect all errors in one go with `… 2>&1 | grep -E "error:" | sort -u`. Expect the commit message to state what remains unverified. The opposite problem also happens: a sandbox with a preinstalled scientific stack can hide a missing dependency ([networkx and numpy](#networkx-layouts-need-numpy-which-a-plain-pip-install-networkx-doesnt-install)).

### Unable to read GitHub

[`qdvc-gtd-eml` UR.3](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-06-24-conversation-with-claude.md#3-user-request) · [`qdvc-nice-mail` UR.13](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#13-user-request) · [`qdvc-nice-mail-windows` UR.3](https://github.com/qdvc-apps/qdvc-nice-mail-windows/blob/05e28c0e2aa5ed9baf3d28172082169e8442ab9a/ai-assistance-logs/2026-09-28-claude.md#3-user-request)

**Symptom:** "API is rate-limited", "the repo blocks automated access", or (in the Windows port) a whole app rebuilt from the README because the source files couldn't be fetched.

**Cause:** Unauthenticated API calls are limited to 60 an hour, and web fetching of GitHub file pages is unreliable.

**Fix:** `git clone` works where the API and page fetching don't. Otherwise, upload a ZIP of the reference repo. If the assistant says it couldn't read a reference, assume its output is a reconstruction.

### Edits that land in the wrong place

[`qdvc-empirics` UR.21](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#21-user-request) · [`qdvc-household-records` UR.25](https://github.com/qdvc-apps/qdvc-household-records/blob/1815df405bd8a02963b195c06945a22a4248ad19/vibe-coding/2026-07-16-claude.md#25-user-request)

**Symptom:** `cannot find 'TextCodingView' in scope` and many more errors like it; in another app, `NameError: name 'os' is not defined`.

**Cause:** A find-and-replace matched the wrong occurrence of two identical lines and removed the end of one type and the start of the next. In the other case, an edit removed the last use of a module whose import had also been removed, then new code used it again.

**Fix:** After scripted edits, check brace balance and that every type is declared once (Swift), or parse each module's AST and flag names used but never imported (Python). `qdvc-household-records` added that test.

### Theorising for rounds instead of measuring

[`qdvc-tadaima` UR.33](https://github.com/qdvc-apps/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#33-user-request) · [`qdvc-gtd-eml` UR.13](https://github.com/qdvc-apps/qdvc-gtd-eml/blob/6d1830abdb77370bc7962e6db6f9f19debecc41c/vibe-coding/2026-07-03-claude.md#13-user-request) · [`qdvc-empirics` UR.10](https://github.com/qdvc-apps/qdvc-empirics/blob/1cbefda7dde360b33c70c014d7e7a8998d5eb149/ai-assistance-logs/2026-11-11-claude-1.md#10-user-request)

**Symptom:** About 12 rounds on the Tadaima CPU spin, four on the Plotly ghosting, five on the Empirics sidebar, and five on Tadaima's thumbnail grid. Each round brought a new confident explanation.

**Cause:** Without being able to run the app, the assistant reasoned from code and reshuffled widgets.

**Fix:** What ended each saga was evidence: the native `py-spy` stack, the browser inspector's element, the pattern of which pages failed, or a known-good sibling app to copy. Ask for that evidence, or for an isolation switch (`--no-filmstrip`, `GSK_RENDERER=cairo`), by the second failed round.

### Confident claims that didn't hold up

[`qdvc-nice-mail` UR.21](https://github.com/qdvc-apps/qdvc-nice-mail/blob/b7119f3ba6fe2c28101819b42af81a96173ba530/vibe-coding/2026-07-13-claude.md#21-user-request) · [`test-management` UR.7](https://github.com/qdvc-apps/test-management/blob/c7886bdb2e25dc943aff8825f8dfc5d1bceac9b6/ai-assistance-logs/2026-08-01-claude.md#7-user-request)

**Symptom:** The assistant implied `Gtk.ImageMenuItem` was "more deprecated" than `GtkUIManager`/`GtkAction` (it isn't; both were deprecated in GTK 3.10), and it named a product "testfold" without being asked.

**Fix:** Asking "where did you get that information?" got an honest retraction. Pattern Z3 (Assumption Audit) and asking for sources on present-day facts both help.

### New code that ignores the codebase's conventions

[`test-management` UR.11](https://github.com/qdvc-apps/test-management/blob/c7886bdb2e25dc943aff8825f8dfc5d1bceac9b6/ai-assistance-logs/2026-08-01-claude.md#11-user-request)

**Symptom:** A new About screen had blank icons, wrong padding, and underlined text on hover.

**Cause:** The icon map's values are complete SVG element strings, but the new entries were bare path data, so the `<svg>` rendered empty without any error. The padding and hover styles skipped existing classes for the same reason.

**Fix:** Ask the assistant to find and follow the existing convention before adding to it, and to check for the whole family of the bug, not just the instances reported.

### Silent fallbacks, and one field meaning two things

[`qdvc-checklists-android` UR.15](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#15-user-request) · [`qdvc-checklists-android` UR.21](https://github.com/qdvc-apps/qdvc-checklists-android/blob/2775795e170ee9db0702a79cc1cb1814f2e74395/vibe-coding/2026-08-01-claude.md#21-user-request)

**Symptom:** Creating an item made its checklist vanish and reappear as a new checklist. Later, tapping a newly created checklist opened a different one.

**Cause:** A `WriteResult.docId` meant "the folder this write touched", which was the checklist for some operations and the new item for others. The caller always treated it as the checklist. Separately, a launch-time snapshot was used as the authority on what exists, and `?: after.firstOrNull()` silently substituted an unrelated checklist when the new one wasn't in it.

**Fix:** Name fields for exactly what they identify. When the current item can't be found, show an empty state rather than another item. The two meanings coincide in the common cases, which is why this kind of bug passes review.

### Parser rules inferred from typical samples instead of the format's definition

Non-public QDVC project (2026-11-25) · [report](reports/processed/2026-11-25-ai-mistakes-parser-rules-from-samples.md)

**Symptom:** An importer for the line-tagged `refer` bibliographic format (`%` + tag letter + value on each line, with EndNote's tagged export adding `%0` for the reference type) split every record of a real, valid file in two. Each title became a record of its own, and the rest of the record, with no title, was skipped as invalid. The file's records began `%T <title>` followed by `%0 <reference type>`.

**Cause:** Confirmed by reproducing with the file and reading the parser. The assistant had added a rule that every `%0` line starts a new record, because most exporters write `%0` first, to cope with hypothetical files that run records together. No specification says that: classic `refer` (the groff `refer(1)` manual page) separates records with blank lines and says nothing about field order, EndNote's import-filter documentation lets the type tag appear anywhere, and bibutils also splits on blank lines. The assistant's own test file was written in the assumed shape, so the tests checked the assumption rather than the format, and it never asked for a real sample. The same session joined repeated single-valued tags, guessing at wrapped values, where the manual says a later occurrence replaces the earlier one (except `%A` and `%E`).

**Fix:** `%0` now starts a new record only when the current record already has one, and blank lines remain the separator. The owner's real sample (redacted) became a regression test, the invented test file was corrected, and the other departures from the manual were listed for the owner to decide on. Confirmed by the owner.

**Watch for:** When writing a parser, note where each rule comes from. Rules from the specification decide structure (where records begin and end, what's required, what repetition means); patterns seen in samples may only add tolerance where the specification is silent, and must never change how valid input is read. "Most files do X" isn't a rule. A guess that changes how input is divided or merged is a question for the human: say what the specification says, what the samples suggest, and ask for a real file. Don't let invented test data confirm your own assumptions.

## Sources

Logs from these repositories were read for this document (as at 19 November 2026): `.github`, `clock`, `focus`, `hernantz-timezones`, `qdvc-android-app-specification`, `qdvc-apa-citations-check`, `qdvc-articulate`, `qdvc-autolitreview-tester`, `qdvc-bibliotheca`, `qdvc-bibliotheca-mac`, `qdvc-case-manager`, `qdvc-cat-android`, `qdvc-checklist-studio`, `qdvc-checklists-android`, `qdvc-citations-checker-mac`, `qdvc-countdowns`, `qdvc-countdowns-android`, `qdvc-csv-android`, `qdvc-datesetter`, `qdvc-display-toggle`, `qdvc-empirics`, `qdvc-equip`, `qdvc-family-noticeboard`, `qdvc-gioia-diagram`, `qdvc-gtd-eml`, `qdvc-gtd-eml-gtk4`, `qdvc-gtd-eml-mac`, `qdvc-health-dashboard`, `qdvc-household-records`, `qdvc-html-to-markdown-SPA`, `qdvc-logbook`, `qdvc-manuscript`, `qdvc-markdown-notebook`, `qdvc-markdown-notebook-android`, `qdvc-markdownpedia`, `qdvc-meetings`, `qdvc-nice-mail`, `qdvc-nice-mail-mac`, `qdvc-nice-mail-windows`, `qdvc-nts-android`, `qdvc-offgrid-buddies`, `qdvc-paperpod-suite`, `qdvc-podcaster`, `qdvc-progress-gtk4`, `qdvc-progress-mac`, `qdvc-python-gtk-app-specification`, `qdvc-quickprisma`, `qdvc-research-pipeline`, `qdvc-response-letter-mac`, `qdvc-super-switcher`, `qdvc-tadaima`, `stickies` and `test-management`.

`qdvc-offgrid-bot`, `qdvc-project-cards` and `qdvc-tidy-transcript` had no logs at the time.

Some repositories already keep their own lists of traps, which go into more depth for their platform and are worth reading alongside this one:

- The `MAINTENANCE.md` "gotchas" sections, notably in `qdvc-display-toggle`, `qdvc-bibliotheca` and `qdvc-markdown-notebook-android`.
- The trap appendix of the [QDVC Android app specification](https://github.com/qdvc-apps/qdvc-android-app-specification).
- The GTK 3 rationale in the [QDVC Python GTK app specification](https://github.com/qdvc-apps/qdvc-python-gtk-app-specification).

## Reporting new cursed knowledge

If you hit a new gotcha while working on a QDVC project, public or not, don't edit this document directly: write a short report and file it in [`reports/inbox/`](reports/inbox/), following [`docs/WRITING-REPORTS.md`](docs/WRITING-REPORTS.md). Non-public projects can report too, under strict rules about what may be included. Maintainers fold reports into this document following [`docs/MAINTENANCE.md`](docs/MAINTENANCE.md).
