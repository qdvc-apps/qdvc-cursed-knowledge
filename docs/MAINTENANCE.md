# Maintaining QDVC Cursed Knowledge

These instructions are for whoever maintains this repo, human or AI. The main job is working through the reports in [`reports/inbox/`](../reports/inbox/) and folding them into the [Cursed Knowledge document](../README.md). How reports are written is described in [`WRITING-REPORTS.md`](WRITING-REPORTS.md); read that too, because the checks below assume its rules.

## 1. Repo layout

- `README.md` is the Cursed Knowledge document itself. It's the only file most readers will open.
- `reports/inbox/` holds reports waiting for review, one gotcha per file.
- `reports/processed/` holds every report that has been reviewed, each with an outcome note at the top. Reports are never deleted, because entries from non-public projects link to them as their only source.
- `reports/TEMPLATE.md` is the template contributors copy.
- `docs/` holds these instructions and the report-writing instructions.
- `ai-assistance-logs/` holds the chat logs of AI sessions on this repo, kept according to the [recordkeeping specification](https://github.com/blairwang-online/vibecoding-recordkeeping-specification). Maintenance sessions done with an AI are logged here like any other.

## 2. Processing the inbox

Start from a fresh copy of the repo, then list `reports/inbox/` (ignore `.gitkeep`) and work through the reports oldest first. For each one, go through the steps below in order.

### Step 1: Privacy screen (non-public reports first)

Do this before anything else. For a report whose **Project visibility** is "Non-public", go through the checklist in section 5 of [`WRITING-REPORTS.md`](WRITING-REPORTS.md) line by line: no project name or identifiers, no description of what the project does, no project-specific symbols, paths, data, screenshots or stack frames, and no link to any private log or repo. Also check public reports for accidental mentions of non-public projects.

If something slipped through:

1. Don't incorporate the report yet.
2. Scrub the file in its own commit, replacing the revealing details with neutral placeholders.
3. Tell the owner of the non-public project straight away. The original text is still in the git history of a public repo, so scrubbing the file isn't enough on its own. Whether to rewrite history (and force-push) is the owner's decision, not the maintainer's.

### Step 2: Check the report is usable

- Every section of the template is filled in, even if only with "Unknown".
- For a public project, the **Log link** works: the file exists at that commit and the anchor matches a `## N. User request` heading in it. You can check with `git clone` and `git cat-file -e <hash>:<log-path>`, then look for the heading. If the link says "Pending" and the log has since been committed, resolve it yourself. If the log still isn't available, leave the report in the inbox and move on.
- Where it's cheap to do so, sanity-check the claimed cause: the library's documentation, its source, or a quick test. Contributors are asked to say how sure they are; carry that through.

### Step 3: Decide the outcome

Search the Cursed Knowledge document for the error text, the API name and the symptom, then choose one:

- **New entry:** a gotcha that isn't in the document.
- **Merged:** the gotcha is already there, and the report adds something (another occurrence, another version affected, a better fix, a correction).
- **Declined:** not cursed knowledge (an ordinary bug in the project's own logic), a duplicate that adds nothing, too vague to be useful even after asking, or a non-public report that can't be made safe. Write the reason down.

If you're unsure whether something qualifies, lean towards including it. The document is only useful if it catches the gotchas people actually hit.

### Step 4: Write or update the entry

Follow the format the existing entries use exactly:

```markdown
### <Title that names the trap>

<link line>

**Symptom:** …

**Cause:** …

**Fix:** …

**Watch for:** …
```

Leave out **Watch for** if there's nothing useful to say.

The link line is how readers check the source. It works like this:

- **Public project:** `` [`qdvc-repo` UR.N](https://github.com/qdvc-apps/qdvc-repo/blob/<full-hash>/<log-path>#n-user-request) ``. Where a fix took several rounds, link the first and last turns with ` → ` between them.
- **Non-public project:** never a log link. Write the date, then a relative link to the report in its final location: `Non-public QDVC project (YYYY-MM-DD) · [report](reports/processed/<file-name>.md)`.
- **Several occurrences:** separate them with ` · `, oldest first. When merging a report into an existing entry, add its link to the end of the line.

Writing style, to keep the document consistent:

- Keep the symptom close to the reporter's wording and include the exact error text, since that's what people search for. For non-public reports, the placeholders stay as placeholders.
- Say whether the fix was confirmed. Use "Confirmed fixed in …" only when the log or report shows it; otherwise "(Not confirmed in the log.)" or "(Not confirmed.)". Mark near misses as "(Caught before shipping.)" and design constraints as "(Design constraint.)".
- Mention versions where they matter, especially for anything that might change in a later release.
- Use the document's spelling (British/Australian: "behaviour", "colour", "centre"), plain words and short sentences.
- When editing, change only the lines that need changing. Don't reflow paragraphs, so diffs stay readable.

Put the entry in the section matching the report's platform, near any related entries. If no section fits, add a new `##` section in a sensible place and add it to the Contents list at the top. Check that the Contents link matches GitHub's anchor for the heading (lowercase, punctuation removed, spaces turned into hyphens).

If a report shows that a platform has since fixed the problem, don't delete the entry. Add "Fixed in <version>" to the Fix part, because older versions are still out there.

### Step 5: File the report as processed

Move the report with `git mv reports/inbox/<file> reports/processed/<file>`, so its history follows it. Then add one line at the very top of the file, above the title:

```markdown
> **Outcome (YYYY-MM-DD):** New entry "<entry title>" in <section>.
```

Or "Merged into "<entry title>" in <section>." Or "Declined: <reason>." Don't otherwise change a report's text, except to scrub it under Step 1.

### Step 6: Update the Sources section

If a public repo has contributed for the first time, add it to the list in the Sources section (the list is alphabetical) and update the "as at" date. Non-public projects are never named there.

## 3. Checks before committing

- Every `TODO` and every "Pending" log link has been resolved.
- Every log link points to a file that exists at the pinned commit, with an anchor that exists in that file.
- Every relative link to `reports/processed/` points to a file that exists.
- Every Contents link matches a heading, and no two headings share the same anchor.
- No non-public project is named or described anywhere in the document.

Then commit, following the recordkeeping specification. For example:

`[AI-assisted] [docs] Incorporate 3 cursed knowledge reports into the GTK 4 and Android sections`

Describe what happened to each report in the commit description (new entry, merged or declined), within the three-sentence limit.

## 4. Pull requests

Contributors are asked not to edit `README.md` directly. If a pull request does, check it the same way as a report: privacy first, then usefulness. Either rewrite its change to match the format, or ask for it to be resubmitted as a report in `reports/inbox/`.

## 5. Occasional sweeps of the public logs

Reports depend on people remembering to write them. Every so often, it's worth sweeping the public repos' AI assistance logs for gotchas nobody reported, the way the document was first built:

1. Clone every repo in the [qdvc-apps](https://github.com/orgs/qdvc-apps/repositories) organisation with `git clone --depth 1`. Use clones rather than the GitHub API, which limits unauthenticated use to 60 requests an hour.
2. Find the log folders (`ai-assistance-logs/`, `ai-assistance/`, `vibe-coding/` or `vibe-coding-logs/`) and the logs added or changed since the last sweep.
3. Split each log into turns at its `## N.` headings. Look for user turns that report a problem (error messages, "doesn't work", "still", "crash", "glitch", "warning") and for assistant turns that explain a cause ("root cause", "turns out", "known issue", "quirk", "workaround").
4. Read the diagnosis and the user's next message, to see whether the fix was confirmed.
5. Write each finding up as a report in `reports/inbox/`, then process it as above. That way entries found by sweeps have a report behind them in `reports/processed/`, just like reported ones.

Repos that are private, or not in the qdvc-apps organisation, are out of scope for sweeps. Their gotchas only arrive through reports.
