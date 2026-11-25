# Parser rules inferred from typical sample files instead of the format's definition

- **Date encountered:** 2026-11-25
- **Reported by:** Claude (Anthropic), Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Mistakes the AI kept repeating
- **Versions:** Not tied to a platform version (a parser-design mistake); Python 3.13; Claude Opus 5.5
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed
- **Related entry:** "Guessing an API instead of reading its source"

## Symptom

An importer for the line-tagged `refer` bibliographic format (each field on its own line as `%` + tag letter + value; EndNote's "tagged" export adds `%0` for the reference type) split every record of a real file in two. The import preview showed each title as a record of its own, followed by a record with everything else but no title, which was then skipped as invalid.

The file was valid. Records were separated by blank lines, as the format requires, but each record began `%T <title>` followed by `%0 <reference type>`.

## Cause

Confirmed by reproducing with the file and reading the parser.

The parser had a rule that every `%0` line starts a new record. The assistant added it because most exporters write `%0` as the first line of each record. It was meant to cope with possible files that run records together without blank lines, though no such file had been seen. No specification says this:

- Classic `refer` (the groff `refer(1)` manual page) defines records as fields one per line, separated by blank lines. It doesn't use `%0` at all, and says nothing about which field comes first.
- EndNote's own import-filter documentation says the reference type tag may appear anywhere in a record's template.
- bibutils, a widely used converter, also splits records on blank lines.

A record separator was already well defined, so the field order of typical samples should never have been used as a second one. The BibTeX equivalent would be starting a new entry at `author =` because exporters usually write it first, even though `@type{…}` marks where entries begin.

Two things made it worse:

1. **The assistant wrote its own test file in the assumed shape**, with a record that started at `%0` and had no blank line before it. The tests passed, but they checked the assumption, not the format.
2. **It didn't ask.** The format definition and the samples disagreed in how much they promised, and the assistant didn't point out the guess or ask the owner for a real sample. It chose the narrower reading itself.

The same session repeated the pattern elsewhere in the parser. When a single-valued tag appeared twice in one record, it joined the two values, guessing the exporter had wrapped a long value. The `refer` manual says later occurrences of fields other than `%A` and `%E` replace earlier ones.

## Fix

`%0` now starts a new record only when the record already has one. Blank lines remain the record separator, as the format defines. The owner's real sample (with values redacted) is now a regression test, and the invented test file was corrected. The owner confirmed that the fix worked.

The other guesses that depart from the manual were listed for the owner to decide on, rather than kept silently.

## Watch for

- **When writing a parser, write down where each rule comes from:** the specification, or patterns seen in samples. Rules from the specification decide structure: where records begin and end, what is required, and what repetition means. Patterns in samples may only add tolerance for input the specification leaves undefined. They must never redefine something the specification already defines.
- **"Most files do X" isn't a rule.** Field order, first lines, header text, and the order that exporters happen to use are habits of particular tools, and the next tool will differ.
- **A guess that changes how the parser divides or merges input is a question for the human,** not something to decide alone. Say what the specification says, what the samples suggest, where the two differ, and ask for a real file.
- **Don't let test data you invented confirm your own assumptions.** Build test files from the specification's examples and from real samples, or label invented ones as assumptions.
- **Each tolerance should cover only a case the specification leaves undefined.** Before adding one for malformed input (odd line endings, wrapped values, stray lines), check that it can't change how valid input is read.

## Minimal example

```python
import re

TAG = re.compile(r"^%(\S)\s?(.*)$")

def records_wrong(text):
    """Overfitted: assumes %0 is always first, so it splits valid records."""
    recs, cur = [], []
    for line in text.splitlines():
        m = TAG.match(line)
        if not line.strip():
            if cur: recs.append(cur); cur = []
        elif m:
            if m.group(1) == "0" and cur:      # <- rule inferred from typical samples
                recs.append(cur); cur = []
            cur.append(m.groups())
    if cur: recs.append(cur)
    return recs

def records_right(text):
    """Per the format: records are separated by blank lines; field order is free."""
    recs, cur = [], []
    for line in text.splitlines():
        m = TAG.match(line)
        if not line.strip():
            if cur: recs.append(cur); cur = []
        elif m:
            cur.append(m.groups())
    if cur: recs.append(cur)
    return recs

sample = "%T A title\n%0 Journal Article\n%A Doe, Jane\n\n%T Another\n%0 Journal Article\n%A Roe, Rick\n"
print(len(records_wrong(sample)))  # 4: each title split from the rest of its record
print(len(records_right(sample)))  # 2
```
