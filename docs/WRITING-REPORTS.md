# Writing a Cursed Knowledge report

These instructions are for anyone working on a QDVC project, human or AI, public or not. If you run into a piece of cursed knowledge, write a short report and file it in this repo's [`reports/inbox/`](../reports/inbox/) folder. A maintainer will fold it into the main [Cursed Knowledge document](../README.md) later, following [`MAINTENANCE.md`](MAINTENANCE.md).

## 1. What counts as cursed knowledge

Write a report when a platform, rather than your own logic, surprised you. For example:

- A build or packaging error caused by a toolchain, SDK, plugin or dependency (Gradle, Xcode, SwiftPM, Flatpak, the .NET SDK, KSP and so on).
- An API or SDK that behaves differently from what its name, its documentation or common sense suggests: a signal that fires earlier than expected, a property whose setter is internal, a widget that ignores a setting.
- A glitch or defect caused by the platform: layout, rendering, focus, insets, theming, clipboard, performance.
- A file format or standard with a non-obvious rule (encodings, line endings, escaping).
- A version change that broke code that used to work (renamed parameters, removed APIs, new analyzer rules).
- A mistake an AI assistant made more than once, or made confidently, that cost real time.

Signs that it's worth reporting: the fix took more than one round, the first explanation turned out to be wrong, the fix is a workaround for a platform limitation, or you thought "I wish someone had told me that".

What doesn't count: ordinary bugs in your own code (a typo, an off-by-one, a missing feature) that would have happened on any platform. If in doubt, write the report anyway and let the maintainer decide.

## 2. Before you write

Search the [Cursed Knowledge document](../README.md) and the files in [`reports/inbox/`](../reports/inbox/) for the error message or the API name.

- If the gotcha is already there and you have nothing new, don't write a report.
- If it's there but you have something new (another version affected, a better fix, a case where the documented fix didn't work, evidence the documented cause is wrong), write a report and name the existing entry under **Related entry**.

## 3. Writing the report

1. Copy [`reports/TEMPLATE.md`](../reports/TEMPLATE.md) into `reports/inbox/`.
2. Name it `YYYY-MM-DD-<platform>-<short-slug>.md`, using the date you hit the problem, a platform code from the table below, and a few lowercase hyphenated words. If the name is already taken, add `-2`.
3. Write about one gotcha per file. Two unrelated problems found in the same session are two reports.
4. Fill in every section of the template. Write "Unknown" rather than deleting a section.
5. Keep the symptom close to how it was first reported, with the exact error text. That is what the next person will search for.
6. Say how sure you are about the cause, and whether the fix was confirmed to work. "Fix not yet confirmed" is a perfectly good status.
7. Give versions: OS, toolkit or SDK, key libraries, toolchain, and the device if it matters. Many gotchas only apply to some versions.

| Platform code | Section of the Cursed Knowledge document |
| --- | --- |
| `gtk3` | Python and GTK 3 (PyGObject) |
| `gtk4` | GTK 4 and libadwaita |
| `linux-desktop` | Linux desktop integration and packaging |
| `python-shell` | Python, the shell and the terminal |
| `macos-swift` | Swift, SwiftUI and AppKit (macOS) |
| `macos-toolchain` | macOS toolchain: Xcode and SwiftPM |
| `android-ui` | Kotlin, Jetpack Compose and Android UI |
| `android-data` | Android storage and data: SAF and Room |
| `android-build` | Android build tooling: Gradle and Kotlin |
| `windows` | Windows, .NET and WinForms |
| `web` | Web: HTML, CSS, SVG, JavaScript and Plotly |
| `other-libs` | Other libraries (llama.cpp, Textual, WeasyPrint, matplotlib, …) |
| `file-formats` | File formats and standards |
| `ai-mistakes` | Mistakes the AI kept repeating |
| `new` | None of these fit; suggest a section name in the **Platform** field |

## 4. Public projects: link to the log

For a project in the public [qdvc-apps](https://github.com/orgs/qdvc-apps/repositories) organisation, the **Log link** is a commit-pinned permalink to the turn in the AI assistance log where the problem was raised:

`https://github.com/qdvc-apps/<repo>/blob/<full-commit-hash>/<log-path>#<n>-user-request`

For example, `…/qdvc-tadaima/blob/14219673b2454a4847a001ba9493ea8170de7f24/vibe-coding/2026-07-25-claude.md#33-user-request`. Use a commit that contains the log file (`git log -1 --format=%H -- <log-path>`), and take the anchor from the heading (`## 33. User request` becomes `#33-user-request`).

If the log hasn't been committed yet, write `Pending: <repo>, <log-path>, UR.<n>` and the maintainer will resolve it. If the problem came up outside an AI session, write "No log (found by hand)" and describe the context in the Symptom section.

## 5. Non-public projects: the two rules

Reports from non-public projects are filed in this public repo, so they become public the moment they're committed, and stay in the git history even if the file is edited later. Two rules apply, with no exceptions:

1. **Reveal nothing about the project.** The report must describe the platform's behaviour, not the project.
2. **No link to the project's AI assistance log**, or to its repo, issues or any other private material. Write "Not available (non-public project)" in the Log link field and "Non-public QDVC project" in the Project field.

Before filing, go through this checklist. Remove or replace with a neutral placeholder (`<MyView>`, `<AppModel>`, `<item>`, `/path/to/workspace`):

- [ ] The project's name, codename, working title, app ID, bundle ID, package name (`qdvc.<something>.android`) and repo name.
- [ ] What the app does, who it's for, and its domain vocabulary. Write "a list of items", not "the list of invoices".
- [ ] The project's own type, function, variable, file and folder names, including in code, error messages and stack traces. Keep the platform's symbols (`Gtk.ListView`, `NavigationSplitView`) and the platform's wording of the error.
- [ ] File paths, user names, machine names, server names, URLs and email addresses.
- [ ] Data of any kind, including sample records.
- [ ] Screenshots and pasted logs. Describe what you saw instead.
- [ ] Stack frames from the project's code. Keep the platform frames, and replace the project's frames with `<app frame>`.
- [ ] Details that only make sense for one project: a feature, a workflow, a client, a timeline.

Then read the finished report and ask: could someone who knows the QDVC apps work out which project this is, or what it does? If yes, generalise further. The **Minimal example** section is the best way to keep the report useful: a few lines of generic code that reproduce the problem with neutral names.

If a date or version would itself be revealing, give only the month (use the first of the month in the file name and add "(month only)" after the date).

An AI writing a report for a non-public project must show the finished report to the human it's working with and get their go-ahead before it's committed. If you're not sure whether something is revealing, leave it out and ask the project owner.

## 6. Filing the report

Commit the new file to `reports/inbox/` in this repo, or open a pull request with it. Don't edit the main Cursed Knowledge document yourself; that's the maintainer's job, so entries stay consistent. Following the [recordkeeping specification](https://github.com/blairwang-online/vibecoding-recordkeeping-specification), a commit message might be:

`[AI-assisted] [docs] Add cursed knowledge report: Gtk.ListView jumps to the top when its model is replaced`

An AI that can't commit should produce the report as a downloadable Markdown file with the correct file name, plus a commit message, for the human to file.

## 7. A standing instruction for AI sessions

To have an AI assistant keep an eye out for cursed knowledge throughout a session (Pattern X5, Standing Order), paste something like this into the conversation, choosing the right final paragraph:

```text
Throughout this conversation, keep an eye out for "cursed knowledge": build errors, API or SDK surprises, platform glitches, file-format traps, or mistakes you repeat, where the platform rather than our own logic was the problem. When one has been fixed (or we've given up on it), say so, and offer to write a report following https://github.com/qdvc-apps/qdvc-cursed-knowledge/blob/main/docs/WRITING-REPORTS.md using the template at https://github.com/qdvc-apps/qdvc-cursed-knowledge/blob/main/reports/TEMPLATE.md. Check the Cursed Knowledge document first so you don't report something already there.

This is a public project in the qdvc-apps organisation, so include a commit-pinned link to the log turn (or "Pending" if the log isn't committed yet).

OR

This is a non-public project. Follow section 5 of the instructions strictly: reveal nothing about this project, use neutral placeholders, include no link to this log or repo, and show me the finished report before I file it.
```
