---
name: pages-cli
description: Use this skill whenever the user wants Claude to open, edit, format, export, or otherwise operate on an Apple Pages document (.pages file), or mentions "Pages", "pages-cli", "Apple Pages", or a document/manuscript/book they're working on in Pages -- even if they don't name the tool explicitly. Also use it whenever the user asks to check formatting consistency, audit styles/fonts, or look for formatting mistakes in any large Pages document (a manuscript, report, book), since the naive approach of querying Pages paragraph-by-paragraph is far too slow for documents of any real size and this skill has the fast, correct method.
---

# Operating Apple Pages via pages-cli

`pages-cli` (https://github.com/marcelrgberger/pages-cli) drives the real Apple Pages
app on this Mac via AppleScript, exposing document, text, table, media, template, and
export operations as a command-line tool. This only works in a session with Bash
access to this machine -- it's not a claude.ai chat file format, and it won't do
anything on other devices or sessions.

## Before doing anything: check it's installed

```bash
ls ~/Code/pages-cli/agent-harness/.venv/bin/pages-cli
```

If that doesn't exist, set it up inside your clone of the repository this skill ships
in, which must be at `~/Code/pages-cli` (this skill is its `skills/pages-cli` folder).
Don't clone the original upstream project for this: it lacks the commands below. The
plugin-marketplace method needs the `claude` CLI, which isn't available in a desktop
session, so install manually:

```bash
cd ~/Code/pages-cli/agent-harness
python3 -m venv .venv && source .venv/bin/activate
pip install -e .
```

To make Claude see this skill, link it into your own skills folder (one time):

```bash
ln -s ~/Code/pages-cli/skills/pages-cli ~/.claude/skills/pages-cli
```

## Using the CLI

Activate the venv, then run `pages-cli`. Global flags (like `--json`, `--document`) go
*before* the subcommand:

```bash
source ~/Code/pages-cli/agent-harness/.venv/bin/activate
pages-cli --json document open "/path/to/file.pages"
pages-cli --json document info
pages-cli --help          # top-level command groups
pages-cli text --help     # e.g. text operations
```

Command groups: `document`, `text`, `table`, `media`, `export`, `template`, `session`,
`repl`. Use `--help` on any of them to see what's available -- don't guess at flags.

The text-editing and offline-reading commands below (`text find`, `text replace*`,
`text delete-range`, `document read-file`, `document save --reopen`) are not in the
original upstream project (marcelrgberger/pages-cli); they are on `main` of this fork,
the repository this skill ships in. If the checkout is on another branch or was
cloned from the original project they will be missing, so check `pages-cli text --help`
shows `find` before relying on them.

## Working rules (they apply to every Pages job)

- **Work only when asked, and only once the person has stopped editing.** Scripted
  changes go straight into the open document and can collide with unsaved work.
- **Say so before opening Pages,** so a window appearing is no surprise.
- **Prefer checking text from the file on disk** (`document read-file`) over querying
  Pages. Nothing is touched and no window opens.
- **Never open a backup in Pages.** Back up with `cp` (or `cp -R` for a bundle) and
  leave the copy alone.
- **Close only the document you opened, and never close every document** -- that can
  discard someone's unsaved work. Use `pages-cli --document "<name>" document close
  --no-save` on the one document, never a bare close that lets Pages prompt and
  re-save. `document open` reports
  `was_open`: if the file was already open, leave it open.
- **Keep each command short.** Every CLI call has a 30-second limit; send long output
  to a file in a scratch folder and delete what you leave there when done.
- **Test on a copy first** when the job is new or large.
- **Pages autosaves a saved document in place, and you cannot undo that by closing.**
  About 10-25 seconds after an edit, Pages writes it into the file; closing with
  saving no afterwards does not revert it (measured on macOS 27.2 / Pages Creator Studio
  15.4; re-check on other versions). So `close --no-save` is not an
  undo, and the `cp` backup is the only reliable one. Work on a copy of the file, not
  the original, unless the person wants the original changed.

## Reading a file without opening Pages

```bash
pages-cli document read-file "<file>.pages" --numbered   # add --other for text boxes, headers, footnotes; --json for structure
```

Needs no running Pages. Table cell text is not included. Paragraphs are separated by
line feeds, but a line feed is not the only paragraph break: Pages also breaks
paragraphs at U+0004 and U+0005 (stored in real documents) and, by test, at U+000B,
U+000C, U+000D and U+2029. `--numbered` and the JSON `paragraphs` list apply that rule,
and the numbering matched live Pages exactly on a 632-paragraph document containing
51 U+0005 and 8 U+0004 characters. (A plain split on line feeds gave 573 paragraphs
there, so never number paragraphs yourself from `body`.) U+2028 soft breaks and U+FFFC
object anchors stay inside their paragraph. Other stored control characters such as
U+000E are left inline, unchecked. It reads the saved file, so unsaved edits in an
open window do not appear, and Pages may not yet have autosaved a recent edit.

## Small text fixes in an existing document

Pages' scripting can replace a word or character in place and the paragraph keeps its
style. Replacing the whole body (`text set`) resets styles, so don't use it for edits.
Scripting cannot set paragraph styles at all, and assigning to a character *range*
corrupts the text -- larger rewrites and style changes go through the person.

```bash
pages-cli --json text find "phrase" --ignore-case       # paragraph + character positions
pages-cli text replace "old" "new" --paragraph 3 --occurrence 1
pages-cli text replace-word 3 2 "new"                    # paragraph, word (Pages' own word boundaries)
pages-cli text replace-char 3 14 ","
pages-cli text delete-range 3 10 20                      # inclusive
```

Each edit re-reads every paragraph and reports a mismatch instead of hiding it. The
commands don't call save, but Pages autosaves (see Working rules). A line feed in the
new text starts a new paragraph in the same style. Text is matched exactly, so a
straight quote will not match a curly one.

Tested on a copy of a bilingual document (632 paragraphs, thousands of combining marks,
mixed fonts and styles inside paragraphs). These results were measured on one setup
(macOS 27.2, Pages Creator Studio 15.4); timings and Pages' behaviour may differ on
other versions, so treat them as observations to re-check, not guarantees:
- Positions are Pages' own characters: they equalled Python's code-point counts in all
  632 paragraphs, including those with combining marks.
- Style survives: an italic run, a bold-italic run shortened by a letter, a right-to-left
  word in its own font replaced by a shorter one, and a lone combining mark deleted all
  kept their fonts and sizes, and later runs shifted correctly.
- A replacement that straddles a style change takes the style of the first character
  of the span; the untouched remainder keeps its own. Replace each styled run
  separately if that matters.
- Speed: about 20 ms per character edited. Edits are sent in batches of 600 so no
  script hits the 30-second limit; 247 matches took about 27 s in all, 1,929 about
  46 s. For big replaces run the command in the background or expect it to be slow.
- If a batch fails midway the error says how many assignments were applied: the
  document is then partly edited, so restore the backup.

Routine: (1) copy the `.pages` file with `cp -R` as the backup; (2) make the edits;
(3) if a check fails, close the document and copy the backup back over the file, then
report; (4) otherwise confirm on disk with `document read-file` (allow up to ~30 s for
the autosave, or `document save`); (5) delete the backup.

## Saving to a new path

`document save --path P` writes the file, but a document that was edited by script
before its first save-as can stay an **Untitled** window that is not linked to the
file (observed on macOS 27.2 / Pages Creator Studio 15.4); opening the file again then
gives a second copy. The command warns when this happens. Use `document save --path P --reopen` to close that window without saving and
open the written file. Pages may leave an extra `Untitled.pages` autosave behind (see
Pitfalls).

## Exporting text and PDF

Export both together from the finished file, to temporary names, and check before
putting them in place:

```bash
pages-cli export pdf "<folder>/<name> export.pdf" --image-quality Best --password "<pw>"   # omit --password if none
pages-cli export text "<folder>/<name> export.txt"
grep -a -c '/Encrypt' "<folder>/<name> export.pdf"       # must be above 0 when a password was set
```

The text export has single line breaks between paragraphs. Compare it with
`document read-file` output, then move the files into place; don't overwrite an
existing export until the new one has passed.

## Building a document from a template

The CLI has no command for this. Fill each placeholder with `set word 1` of its
paragraph so the new text takes that placeholder's style; setting `body text` directly
puts everything in the first style. Prepare the body text in a scratch folder with
exactly one line feed between paragraphs and curly quotes, and report any quote that
can't be settled (a leading apostrophe such as 'cause or 'em would become an opening
quote). For a template with a title in paragraph 1 and body in paragraph 2:

```applescript
set body to read (POSIX file "<scratch>/body.txt") as «class utf8»
tell application "Pages"
	set d to make new document with properties {document template:template "<template>"}
	delay 2
	set word 1 of paragraph 2 of body text of d to body
	delay 1
	set word 1 of paragraph 1 of body text of d to "<title>"
	delay 1
	save d in POSIX file "<target>.pages"
	delay 1
	close d saving no
end tell
```

Then check the paragraph count and the saved text on disk (`document read-file`), and
never overwrite an existing file until the new one has passed.

## Pitfalls

- `set bt to body text of d` gives plain text, so `font` and similar fail on it. Use
  `paragraph i of body text of d` directly, or a reference to it.
- Pages won't report paragraph style names. To check formatting, compare `font` and
  `size` of a few paragraphs against a known good file.
- `row` is a reserved word inside `tell application "Pages"`.
- Always guard `open` returning `missing value` (the CLI's `document open` does).
- Check whether Pages is running with `pgrep -x Pages`, not System Events, which may
  not be authorised.
- In zsh `timeout` doesn't exist and `log` is a built-in.
- Text copied back out of Pages may lose blank lines between paragraphs; that's an
  artefact, not an edit.
- **Pages autosaves unsaved documents into the iCloud Pages folder**
  (`~/Library/Mobile Documents/com~apple~Pages/Documents`) as `Untitled N.pages`, even
  if you close with saving no. List that folder before and after any job that creates
  documents, and move only the files you created to the Trash.
- **Pages renames unsaved documents from their content** (a new document containing
  "The cat sat on the mat" became "Cat on Mat"), so targeting an unsaved document by
  name breaks mid-job. Use `front document`, or save it first and use its file name.

## Checking formatting consistency in a large document

This is the part that isn't obvious, and it matters a lot once a document has more
than a few dozen paragraphs.

**Don't** loop `pages-cli text get-font --paragraph N` / `get-font-size` / `get-color`
once per paragraph to audit a whole document. Each call is a separate AppleScript
round-trip through `osascript`, and a document with a few hundred paragraphs needs on
the order of a thousand-plus individual calls that way -- slow, and liable to time out
or flake partway through.

**Do** export the document once and analyze the export with a script:

```bash
pages-cli --json export word "/tmp/check.docx"
```

Then set up a throwaway venv with `python-docx` (the system Python here is externally
managed -- PEP 668 -- so don't `pip install --break-system-packages` into it):

```bash
python3 -m venv /tmp/pages-analysis-venv
source /tmp/pages-analysis-venv/bin/activate
pip install python-docx
python3 ~/.claude/skills/pages-cli/scripts/analyze_formatting.py "/tmp/check.docx"
```

That script (bundled with this skill) walks every paragraph and run in the exported
docx and reports:
- font/color usage overall
- **font size broken down per paragraph style** -- this is the real consistency
  check. A style having its own fixed size (headings bigger than body, etc.) is
  normal; a style whose size *varies* across its own instances is the actual signal
  of a mistake.
- any paragraph that mixes more than one font or size within itself

On a book-length document this analyzed over a thousand text runs in one pass, versus
over a thousand individual CLI calls the naive per-paragraph approach would have
needed.

Clean up the export and the throwaway venv when done (they're scratch artifacts, not
something to leave lying around):

```bash
rm -rf /tmp/check.docx /tmp/pages-analysis-venv
```

Only fall back to live per-paragraph `pages-cli` queries when the document is small,
or when the check specifically needs the live/unsaved state of the open document
(the exported copy is a snapshot as of export time).

## Reading the results: investigate anomalies, don't just flag counts

When a style's usage count looks like it doesn't add up -- e.g. a "no first-line
indent" style shows up more times than there are section openings it's supposedly
for -- resist the urge to immediately call that an inconsistency. Pull up the
surrounding paragraphs and read them in context first.

The reason this matters: in one real case, a no-first-line-indent body style (used for
the paragraph right after each section heading) showed up on one extra paragraph that
wasn't a heading. Reading the surrounding text showed it was
the paragraph resuming a sentence that had been interrupted by an inset quotation --
the same typographic reason for dropping the indent after a heading applies after a
block quote too. It was a deliberate, coherent choice, not a mistake. A pure
count-based check would have wrongly flagged it.

So: use the script's output to find *candidates* worth a look, then actually read
the paragraph text (and what precedes/follows it) before reporting something as a
formatting problem to the user.
