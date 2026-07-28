# CI build — what to fix

Diagnosis of the failing `Compile LaTeX and Publish PDF` workflow on `feature/github-ci`.

---

## 1. The actual blocker: `arydshln` breaks `\hline`

```
./contents/0-auxiliary/chapter-intros/chapter-1.tex:37: Extra }, or forgotten $.
<recently read> \egroup
l.37     \end{tabularx}
==> Fatal error occurred, no output PDF file produced!
```

### The error is not on line 37

Line 37 is just `\end{tabularx}`. **`tabularx` re-typesets its whole body** to measure the
`X` column, so every error inside the table gets reported at the closing line. The real
problem is somewhere in lines 20–36.

(Verified: put a deliberately broken macro on line 13 of a `tabularx` and TeX still reports
it at `\end{tabularx}`.)

### Root cause

`config/zkdl-template.cls` line 64 loads `arydshln`, which **redefines `\hline`**:

```tex
\def\hline{\noalign{\ifnum0=`}\fi
        \ifadl@zwhrule \vskip-\arrayrulewidth
        ...
```

That `\ifnum0=`}\fi` is a brace-balancing hack. Inside `tabularx` — which re-reads the body —
combined with `colortbl`'s `\rowcolor`, the braces stop matching and TeX reports
`Extra }, or forgotten $`.

### Why it appeared now

Commit `e2cd934` ("what if we give a nonstopmode to CI, huh") changed `\hdashline` → `\hline`
in all four chapter-intro tables. `\hdashline` is `arydshln`'s own command and works fine;
`\hline` is the one it monkey-patches. The change swapped a working command for a broken one.

### Fix

`arydshln` is now **completely unused** — after `e2cd934` there is not a single `\hdashline`
or `\cdashline` left in `contents/`. So just stop loading it.

In `config/zkdl-template.cls`, line 64:

```diff
-\RequirePackage{arydshln}
+% arydshln removed: it redefines \hline in a way that breaks tabularx + \rowcolor,
+% and no \hdashline/\cdashline remain in the book.
```

`\Xhline` is unaffected — it comes from `makecell`, not `arydshln`.

**Verified:** with that one line removed, a clean first pass (no `.aux`, exactly like CI)
builds the full book — 307 pages, zero `Extra }` errors.

### Alternative, if you ever want dashed rules back

Keep `arydshln` and revert the four files to `\hdashline`:

```bash
sed -i '' 's/^\( *\)\\hline$/\1\\hdashline/' contents/0-auxiliary/chapter-intros/chapter-*.tex
```

Removing the package is cleaner, since nothing in the book uses dashed lines.

---

## 2. Why it built locally but not in CI

Your local `book.log`:

```
This is pdfTeX, Version 3.141592653-2.6-1.40.27 (TeX Live 2025)
Package: minted 2025/03/06 v3.6.0
Output written on book.pdf (305 pages)
```

Two differences from CI, both of which mask the bug locally:

1. **You build incrementally.** `book.aux`, `book.toc`, `book.bbl` all exist in your working
   tree, so references resolve on the first pass. CI starts from a clean checkout.
2. **CI ran with `-halt-on-error`.** Because the workflow set neither `compiler` nor `args`,
   `entrypoint.sh` fell back to its defaults, which include `-halt-on-error`. One recoverable
   error therefore killed the whole run.

To reproduce CI locally before pushing:

```bash
rm -f book.aux book.toc book.out book.bbl && find contents -name '*.aux' -delete
pdflatex -interaction=nonstopmode -file-line-error -halt-on-error book.tex
```

---

## 3. Workflow changes — DONE

Already applied to `.github/workflows/build-pdf.yml`.

| # | Problem | Fix |
|---|---------|-----|
| 1 | Failures showed only `exit code 12` | Added `Show LaTeX errors` step (greps `book.log`) + `latex-logs` artifact upload |
| 2 | `texlive_version: 2026` — differs from your local toolchain | `2025`, matching the TeX Live you build with |
| 3 | `pip install --break-system-packages latexminted` | Removed — TeX Live ships its own `latexminted`; the pip copy shadows it in `PATH` |
| 4 | `extra_system_packages: "py3-pygments py3-pip"` | `"python3 py3-pygments"` — minted 3's backend is a Python script and `texlive-alpine` has no Python |
| 5 | `args` unset → silent `-halt-on-error` | `args` set explicitly, without `-halt-on-error` |
| 6 | Release published on every branch push | `if: github.ref == 'refs/heads/main'` |
| 7 | `actions/checkout@v4` — Node 20 deprecation warning | `actions/checkout@v5` |

Items 2–4 were not the cause of this failure, but they were real latent problems: `minted` is
the only package in the build with an external runtime dependency (`minted 3` →
`latexminted` → Python 3 → Pygments), used in 28 blocks in
`contents/4-sumcheck-lookups/1-sumcheck.tex`.

---

## 4. Minor: Cyrillic `ї` in "naïve"

Not a blocker — it compiles today. Worth fixing anyway.

**File:** `contents/3-zk-foundations/8-bulletproofs.tex`, lines 36, 37, 79, 660

The word uses the Ukrainian **`ї` (U+0457)** instead of Latin **`ï` (U+00EF)**. It only works
because the class loads the Cyrillic `T2A` encoding; drop `T2A` and the build breaks. It also
renders a Cyrillic glyph inside English words and leaks into a PDF bookmark.

```bash
sed -i '' 's/naїve/naïve/g; s/Naїve/Naïve/g' contents/3-zk-foundations/8-bulletproofs.tex
grep -rnP '[\x{0400}-\x{04FF}]' contents/ --include='*.tex'   # should print nothing
```

(drop the `''` after `-i` on Linux)

---

## 5. Checked and clean — no action needed

- **Image paths** — every `\includegraphics` reachable from `book.tex` resolves; no missing
  files, no case mismatches (the classic macOS→Linux CI killer).
- **`.gitignore`** — line 306 ignores `/contents/**/*.pdf`, but all figure PDFs are committed
  anyway, so CI checkouts get them.
- **`\subfile` structure** — all subfiles have proper `\documentclass{subfiles}` +
  `\begin{document}`/`\end{document}`.
- **Environment and brace balance** — clean across all content files.
- **Typographic Unicode** — curly quotes, en/em dashes and one U+2010 non-breaking hyphen in
  `3-sumcheck-toolkit.tex` and `1-security-basics.tex`. Harmless, but invisible in an editor
  if you ever want to normalise them.
