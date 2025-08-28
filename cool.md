Let's add numbers to links in text . For example, if link refers to link `2` not only provide a keyword like `Microsoft` but also its actual number in #sym:## Further read section

Here’s a safe, one-pass regex you can run in VS Code to show the numeric reference next to each in-text link.

Steps (Windows):
1. Press Ctrl+Shift+H (Replace in Files), enable “Use Regular Expression (.*)”.
2. Files to include: d:\Prgrms\termius-usage\RDP setup.md
3. Find:
```
\[([^\]]+?)\]\[(\d+)\](?!\s*\[\2\])
```
4. Replace with:
```
[$1][$2] [$2]
```
5. Replace all
What this does:
- Converts e.g. ([Microsoft Learn][3]) → ([Microsoft Learn][3] [3])
- Uses the existing numbering that already maps to your “Further read” section.
- The (?! … ) guard prevents adding the number twice if you run it again.

Example before/after:
- Before: Termius’ docs show the Port Forwarding wizard flow. ([termius.com][2])
- After: Termius’ docs show the Port Forwarding wizard flow. ([termius.com][2] [2])

It works because the regex targets exactly your reference-style links and appends the number once, without touching anything else.

- Matches only reference-style links: `[Label][3]`
- `\[([^\]]+?)\]` captures the label (Group 1) up to the first closing bracket.
- `\[(\d+)\]` captures the numeric reference (Group 2).
- Idempotent guard: `(?!\s*\[\2\])` is a negative lookahead that refuses a match if the same number already appears right after the link (so re-running won’t duplicate).
- Replacement: `[$1][$2] [$2]` keeps the original link intact (`[Label][3]`) and adds a visible number after it (`[3]`), leveraging your existing numbered reference list.
- Safe scope: it won’t touch inline links like `[Text](https://...)`, named refs like `[Text][ref]`, or your “Further read” list items, since they’re not `[Text][number]`.
- VS Code’s regex engine supports lazy quantifiers, groups, backreferences, and lookaheads, so the pattern runs reliably across the file.