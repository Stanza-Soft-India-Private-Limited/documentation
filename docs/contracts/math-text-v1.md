# math-text v1 — normative contract

Status: NORMATIVE. Version 1, 2026-11-11. Design context:
[`docs/superpowers/specs/2026-11-11-math-text-and-language-flags-design.md`](../superpowers/specs/2026-11-11-math-text-and-language-flags-design.md).

One spec, four implementations — backend TS (`src/common/math/`), mobile Kotlin (`core/richtext/`), web TS
(`src/lib/richtext/`), SME portal TS (`admin-ui/src/lib/richtext/`). Every implementation copies
[`math-vectors.json`](./math-vectors.json) **byte-for-byte** and asserts every vector in a unit test. When this
document and a vector disagree, that is a bug in one of them: fix it here, re-version if behaviour changes,
and re-copy the vectors into all four repos in the same wave.

The words MUST / MUST NOT / SHOULD are used in their RFC 2119 sense.

---

## 1. Where maths may appear

Maths is written as LaTeX **inline in the existing text fields** (question stem, options, explanation,
approach, model answer, quiz text, chat answers …). There are no new columns. Text with no maths is
untouched by every stage of this contract.

**Fast path.** A text that contains none of the three sequences `$`, `\(`, `\[` has no maths. Every
implementation MUST return it unchanged (one text segment / identical plain text / valid) without further
work. Servers that rewrite response bodies (old-client fallback) MUST return such bodies byte-identical.

---

## 2. Delimiter grammar — `extractMathSegments(text)`

### 2.1 Definitions

- *Character* — one UTF-16 code unit. All delimiters are ASCII, so scanning by code unit is safe for every
  script (Devanagari, emoji surrogate pairs are never split because no delimiter is a surrogate).
- *Whitespace* — exactly these code units: U+0020 space, U+0009 tab, U+000A LF, U+000D CR, U+000B VT,
  U+000C FF, U+00A0 NBSP. Nothing else counts (implementations MUST NOT use a locale or Unicode
  `isWhitespace`).
- *Newline* — U+000A or U+000D.
- *Digit* — ASCII `0`–`9` only.

### 2.2 The scanner

The extractor scans the text left to right with an index `i`, building segments. At each `i` the **first**
matching rule below applies (this order is the precedence):

| # | At `i` the text starts with | Action |
|---|---|---|
| 1 | `\$` | Escaped dollar. Append a literal `$` (one character, the backslash is consumed) to the current text. `i += 2`. |
| 2 | `\[` | **Line-anchored** display (ruling 21). Succeeds only when `\[` is the first non-whitespace on its line, the first `\]` after `i+2` exists (may be on a later line), and that `\]` is the last non-whitespace on its line. Success ⇒ emit it, `i` moves past `\]`. Failure ⇒ append `\[` to text, `i += 2`. |
| 3 | `\(` | Try an **inline** span closed by the first `\)` after `i+2`; no newline may occur between the delimiters. Success ⇒ emit, `i` moves past `\)`. Failure ⇒ append `\(` to text, `i += 2`. |
| 4 | `$$` | Try a **display** span closed by the first `$$` after `i+2` whose first `$` is not immediately preceded by `\` (may cross newlines). Success ⇒ emit, `i` moves past the closing `$$`. Failure ⇒ append `$$` to text, `i += 2`. |
| 5 | `$` | Try an **inline dollar** span (§2.3). Success ⇒ emit, `i` moves past the closing `$`. Failure ⇒ append `$` to text, `i += 1`. |
| 6 | anything else | Append the character to text, `i += 1`. |

*Line* (rule 2): the characters between two newlines (or the text's start/end); "first/last non-whitespace
on its line" means every other character between the delimiter and that line boundary is whitespace (§2.1).
**Why rule 2 is anchored:** live text writes markdown-escaped brackets mid-sentence
(`… member of 21 \[the Armed Forces of the Union\] engaged …`, a legal-text stem); a mid-line `\[…\]` is
therefore plain text, never maths. `$$…$$` and `\(…\)` are not anchored.

A span (rules 2–5) **fails** when no closer is found under its rule, or when its content (the characters
strictly between opener and closer) is empty or whitespace-only. A failed opener is plain text and
scanning resumes right after it — "a `$` that opens but never closes under the rules is plain text".

Only `\$` is an escape the extractor knows. Every other backslash in text (`\*`, `\\`, `\frac` outside
delimiters …) is passed through untouched for the downstream renderer.

### 2.3 Inline `$…$` (pandoc `tex_math_dollars`, strict)

An inline dollar span opens at a `$` at index `i` when **all** hold:

- the next character `text[i+1]` exists, is not whitespace, and is not `$` (a `$$` is rule 4);

and it closes at the **first** index `j > i+1` such that:

- `text[j]` is `$`;
- `text[j-1]` is not whitespace and is not `\` (a `\$` inside maths is content, never a closer);
- `text[j+1]` is not a digit (end of text is fine);
- no newline occurs in `text[i+1 .. j-1]`.

A `$` inside the search window that does not satisfy the closer test is simply content and the search
continues. If the search reaches a newline or the end of the text first, the opener fails.

Consequences (all pinned by vectors):

- `$2 billion` — no closer ⇒ text. `$100 and $50` — the second `$` is preceded by a space, the search
  hits the end ⇒ both `$` are text.
- `$24,000$`, `$Q$`, `$5$` — maths.
- `$ x$` (space after opener) and `$x $` (space before closer) ⇒ text.
- `$5$10` — the candidate closer is followed by a digit ⇒ not a closer ⇒ text.
- `US$5`, `M $ N`, `1$5` ⇒ text.

### 2.4 Nesting and overlap

There is **no nesting**. Once a span is open its content is opaque until its own closer: `$a \(b\) c$` is
one inline span whose value is `a \(b\) c`; `$$ a $ b $$` is one display span whose value is ` a $ b `.
Overlap cannot happen because the scanner commits to the first successful span in text order and
precedence order, and resumes after its closer.

### 2.5 Segments

`extractMathSegments(text) → Segment[]` with `Segment = { kind: "text" | "inline" | "display", value: string }`.

- `text` value — source characters with each `\$` replaced by `$`; nothing else altered.
- `inline` / `display` value — the exact source characters between the delimiters, **not trimmed**
  (`$$\n x \n$$` ⇒ value `"\n x \n"`).
- Adjacent text pieces are merged into one segment; a text segment is never empty. Maths segments are
  never merged.
- The empty string yields `[]`.
- `$…$` and `\(…\)` produce `inline`; `$$…$$` and `\[…\]` produce `display`. Which delimiter was used is
  not recorded.

Extraction runs **before** any markdown or inline (bold/italic/escape) parsing; maths values never reach a
markdown parser. A renderer that feeds `text` segments back into a markdown parser MUST stop that parser
from treating `$` as maths (escape it, or disable the parser's maths extension), because a text segment can
legitimately contain `$` (currency, `\$`).

Rendering: inline spans flow with the text in the surrounding colour and size; display spans are a block.
**A render failure anywhere shows the maths source in monospace — never blank, never a crash.**

---

## 3. Plain text — `mathToPlainText(latex)` and `mathTextToPlain(text)`

`mathTextToPlain(text)` = concatenation over `extractMathSegments(text)` of: text segments verbatim,
maths segments as `mathToPlainText(value)`. No delimiters survive; nothing is added around display maths
(the source's own newlines, if any, sit outside the delimiters and are kept). It is used for clients below
the maths-capable version, notification/share text, list previews and accessibility labels.

`mathToPlainText(latex)` is a deterministic, dependency-free conversion. Readable Unicode where safe; the
LaTeX source otherwise.

### 3.1 Tokens

| Token | Source form |
|---|---|
| control word | `\` followed by one or more ASCII letters `[A-Za-z]+` (greedy) |
| control symbol | `\` followed by exactly one non-letter character (`\%`, `\,`, `\{`, `\\` …); a trailing lone `\` at end of input is a char `\` |
| group | `{` … matching `}` (balanced; `\{` and `\}` are control symbols, not braces) |
| `^`, `_` | scripts |
| whitespace | a run of whitespace (§2.1) |
| char | any other single character |

**Structural fallback.** If the braces are unbalanced, or a command or script is missing a required
argument (end of input or a `}` where an argument should start), the whole result is the source with
leading/trailing whitespace removed. Nothing is partially converted in that case.

### 3.2 Arguments

A command argument or script argument is read by skipping whitespace, then taking ONE item: a group (its
contents), a control word/symbol (that single command, unconverted args not consumed), or a single char.
`\frac12` ⇒ `1/2`; `x^23` ⇒ `x²3` (TeX semantics). Arguments are converted recursively, and the converted
argument is whitespace-collapsed and trimmed before any rule below looks at it (`x^{ 2 }` ⇒ `x²`).
The optional `[k]` of `\sqrt` is read up to the first `]` (missing `]` ⇒ structural fallback).

### 3.3 Conversion

Output is built left to right:

- **whitespace** ⇒ one space.
- **char** ⇒ itself, except `~` ⇒ space.
- **group** (not an argument) ⇒ its converted contents (braces dropped).
- **control words** ⇒ the table in §3.4.
- **control symbols**: `\%`⇒`%`, `\$`⇒`$`, `\&`⇒`&`, `\#`⇒`#`, `\_`⇒`_`, `\{`⇒`{`, `\}`⇒`}`,
  `\|`⇒`‖`, `\,` `\;` `\:` `\ ` `\\`⇒space, `\!`⇒nothing. Any other control symbol ⇒ its source.
- **scripts** — see §3.5.
- **unknown control word** ⇒ its source verbatim, together with every group that follows it *immediately*
  (no whitespace in between), also verbatim: `\overline{AB}` ⇒ `\overline{AB}`,
  `\begin{pmatrix}a\end{pmatrix}` ⇒ `\begin{pmatrix}a\end{pmatrix}`.
  A name is "in a table" only when the table itself defines it: in JS/TS an object-literal lookup must be
  own-property only (`\constructor`, `\toString`, `\mathbb{toString}` otherwise hit `Object.prototype` and
  print function source). Vectors `prototype-name-control-word`, `prototype-name-mathbb-arg` (B1f).

Finally every run of whitespace in the output is collapsed to one space and the result is trimmed.
Spaces written in the source are kept (`a \times b` ⇒ `a × b`, `a\times b` ⇒ `a× b`): the converter
never invents or swallows spaces around operators.

### 3.4 Control-word table

Structural commands:

| Command | Output |
|---|---|
| `\frac{n}{d}`, `\dfrac`, `\tfrac` | `N/D`, where N and D are the converted arguments, each wrapped in `(…)` when it contains a space or an operator character (§3.6) |
| `\sqrt{x}` | `√X`; X wrapped in `(…)` unless it is a single character or consists only of ASCII digits and `.` |
| `\sqrt[k]{x}` | as `\sqrt`, prefixed by: `∛` for k=`3`, `∜` for k=`4`, otherwise the superscript form of k (§3.5) when every character maps, else `(k)`; then `√` is omitted for 3/4 (`\sqrt[3]{8}` ⇒ `∛8`, `\sqrt[n]{x}` ⇒ `ⁿ√x`, `\sqrt[ab]{x}` ⇒ `(ab)√x`) |
| `\binom{n}{k}` | `C(N,K)` |
| `\text`, `\textrm`, `\textbf`, `\textit`, `\mathrm`, `\mathbf`, `\mathit`, `\mathsf`, `\mathtt`, `\operatorname`, `\ce` | the argument's **raw source**, braces removed, not converted (`\text{km}` ⇒ `km`, `\ce{H2O}` ⇒ `H2O`) |
| `\mathbb{X}` | `ℕ ℤ ℚ ℝ ℂ` for `N Z Q R C`; otherwise the raw argument |
| `\left`, `\right`, `\bigl`, `\bigr`, `\big`, `\Big`, `\bigg`, `\Bigg`, `\displaystyle`, `\textstyle` | dropped. After `\left`/`\right` a `.` delimiter is dropped too |

Symbols (control word ⇒ output):

| Group | Mapping |
|---|---|
| Greek lower | `\alpha`α `\beta`β `\gamma`γ `\delta`δ `\epsilon`ε `\varepsilon`ε `\zeta`ζ `\eta`η `\theta`θ `\vartheta`ϑ `\iota`ι `\kappa`κ `\lambda`λ `\mu`μ `\nu`ν `\xi`ξ `\pi`π `\varpi`ϖ `\rho`ρ `\varrho`ϱ `\sigma`σ `\varsigma`ς `\tau`τ `\upsilon`υ `\phi`φ `\varphi`φ `\chi`χ `\psi`ψ `\omega`ω |
| Greek upper | `\Gamma`Γ `\Delta`Δ `\Theta`Θ `\Lambda`Λ `\Xi`Ξ `\Pi`Π `\Sigma`Σ `\Upsilon`Υ `\Phi`Φ `\Psi`Ψ `\Omega`Ω |
| Binary operators | `\times`× `\cdot`· `\div`÷ `\pm`± `\mp`∓ `\ast`∗ `\circ`∘ `\cup`∪ `\cap`∩ `\setminus`∖ |
| Relations | `\le`≤ `\leq`≤ `\ge`≥ `\geq`≥ `\ne`≠ `\neq`≠ `\approx`≈ `\equiv`≡ `\sim`∼ `\simeq`≃ `\cong`≅ `\propto`∝ `\lt`< `\gt`> `\ll`≪ `\gg`≫ `\in`∈ `\notin`∉ `\subset`⊂ `\subseteq`⊆ `\supset`⊃ `\supseteq`⊇ `\parallel`∥ `\perp`⊥ `\mid`\| |
| Arrows | `\to`→ `\rightarrow`→ `\longrightarrow`→ `\leftarrow`← `\gets`← `\Rightarrow`⇒ `\implies`⇒ `\Leftarrow`⇐ `\leftrightarrow`↔ `\Leftrightarrow`⇔ `\iff`⇔ `\mapsto`↦ `\uparrow`↑ `\downarrow`↓ `\rightleftharpoons`⇌ |
| Big operators | `\sum`∑ `\prod`∏ `\int`∫ `\iint`∬ `\oint`∮ |
| Function names | `\lim` `\sin` `\cos` `\tan` `\cot` `\sec` `\csc` `\log` `\ln` `\exp` `\min` `\max` `\det` `\gcd` ⇒ the name without `\` |
| Misc | `\infty`∞ `\partial`∂ `\nabla`∇ `\forall`∀ `\exists`∃ `\neg`¬ `\emptyset`∅ `\varnothing`∅ `\angle`∠ `\triangle`△ `\therefore`∴ `\because`∵ `\ldots`… `\dots`… `\cdots`⋯ `\vdots`⋮ `\prime`′ `\degree`° `\hbar`ℏ `\ell`ℓ `\langle`⟨ `\rangle`⟩ `\lbrace`{ `\rbrace`} `\vert`\| `\lvert`\| `\rvert`\| `\quad` space `\qquad` space |

Anything not listed is an *unknown control word* (§3.3).

### 3.5 Scripts `^` and `_`

Let S be the converted argument.

1. S empty ⇒ nothing.
2. Every character of S has a glyph in the script's map ⇒ the mapped characters.
   - Superscript map: `0`⁰ `1`¹ `2`² `3`³ `4`⁴ `5`⁵ `6`⁶ `7`⁷ `8`⁸ `9`⁹ `+`⁺ `-`⁻ `−`⁻ `=`⁼ `(`⁽ `)`⁾ `n`ⁿ `i`ⁱ
   - Subscript map: `0`₀ `1`₁ `2`₂ `3`₃ `4`₄ `5`₅ `6`₆ `7`₇ `8`₈ `9`₉ `+`₊ `-`₋ `−`₋ `=`₌ `(`₍ `)`₎
   (deliberately small: these glyphs render on every Android/iOS system font.)
3. Superscript only: S is `∘` (from `\circ`) ⇒ `°`; S is one or more `′` ⇒ S bare (`45^\circ` ⇒ `45°`,
   `f^{\prime}` ⇒ `f′`).
4. Otherwise `^` / `_` followed by S, with S wrapped in `(…)` when it is longer than one character
   (`e^{x}` ⇒ `e^x`, `e^{2x}` ⇒ `e^(2x)`, `\sum_{i=1}^{n}` ⇒ `∑_(i=1)ⁿ`).

### 3.6 Operator characters (fraction wrapping)

`+ - − × · ÷ ± ∓ = ≠ < > ≤ ≥ ≈ /` and space. A fraction part containing any of them is wrapped:
`\frac{a+b}{2}` ⇒ `(a+b)/2`, `\frac{1}{2x}` ⇒ `1/2x`, `\frac{a}{b c}` ⇒ `a/(b c)`.

---

## 4. Validation — `validateMathText(text, { allowlist?, excluded? })` (backend; portal/web pre-check stage 1 only)

Stages run in order and stop at the first failure.

0. **Form feed.** Any U+000C anywhere in the text (inside or outside maths) fails with reason
   ``contains a form feed (U+000C) — `\frac` inside hand-built JSON became a form feed; escape it as `\\frac` ``.
   (JSON `"\frac"` decodes to FF + `rac`.)
1. **Balance / delimiters.**
   - an opener of rule 2–4 that fails under §2.2 (no closer, or empty content) fails with
     ``unclosed or empty `$$` `` / ``unclosed or empty `\[` `` / ``unclosed or empty `\(` ``. An unclosed single `$` is **not** an error
     (it is currency text by §2.3). For `\[` only a **line-anchored** opener with no `\]` after it, or with
     empty content, is an error; a mid-line `\[`, or an anchored one whose `\]` is followed by more text on its
     line (`\[1\] Reference`), is ordinary text and valid.
   - inside each maths span, `{`/`}` (not counting `\{`/`\}`) must balance: ``unbalanced braces``.
2. **KaTeX strict parse** of every maths span (`displayMode` = span kind, `throwOnError: true`,
   `strict` = every strict-mode code is an error **except** `unknownSymbol` / `symbolNotInFont` — a Unicode
   symbol such as `–`, `”`, `₹` inside maths, which the engine draws via its font fallback (B1g, 2026-11-11));
   the reason is KaTeX's message without its `KaTeX parse error: ` prefix. Still errors: `$` or a nested `\(`
   in math mode, `&` outside an alignment environment, a `%` comment running to the closer
   (`commentAtEnd`), Unicode LETTERS such as Devanagari (`unicodeTextInMathMode`).
   Core KaTeX has no `mhchem` but the engine does, so before this parse every `\ce{…}` / `\pu{…}` with a
   balanced argument is replaced by `{}` (the chemistry argument is not judged by math-mode rules); a `\ce`
   without a group still fails here.
3. **Engine exclusions** (rulings 17 + 23 as the engine's DOCUMENTED support; B1g, 2026-11-11) — a control
   word used in a maths span that is in `MATH_ALLOWLIST_EXCLUDED` fails with ``unsupported command `\foo` ``:
   macro definition/expansion (`\def`, `\newcommand`, `\let` …), links/images/html (`\href`, `\url`,
   `\includegraphics`, `\html*`, `\class`, `\id`, `\style`) and RaTeX-README-unsupported commands. Nothing
   else is rejected here: a name KaTeX accepts is rendered by the engine (an unknown one already failed stage 2).
   Applied by default; skipped only explicitly (`{ excluded: null }`).

   **Warnings (non-blocking).** The measured suite — every name in a RaTeX-CMP 0.1.14 golden-suite formula
   scored ≥ 0.70 in [`ratex-support-cases-0.1.14.json`](./ratex-support-cases-0.1.14.json), minus the
   exclusions, generated into `src/common/math/math-allowlist.ts` by `scripts/math/build-allowlist.ts` — is
   kept as DATA. A name outside it (`\tan`, `\tau`, `\max`, `\sup` were absent from the suite) adds
   ``command `\foo` is outside the engine's measured suite`` to `warnings` on an otherwise valid result.
   `{ allowlist: null }` turns warnings off. No write response has a shape for them today, so the API logs
   them at debug and answers exactly as before (never a 400).

The fast path (§1) applies after stage 0: no `$`, `\(`, `\[` ⇒ valid.

Result: `{ ok: true }` (plus `warnings: string[]` when there are any) or `{ ok: false, reason, span, index, message }`:

- `span` — the offending maths source **including its delimiters** (for an unclosed opener: the opener and
  the rest of its line), truncated to 60 characters with a trailing `…`. Stage 0: up to 10 characters each
  side of the form feed, with every form feed shown as the two characters `\f`.
- `index` — the UTF-16 offset in the text where the span starts (stage 0: of the form feed itself).
- `message` = `<reason> near <span>`.

### 4.1 API error shape

A write route that rejects text answers **400** with

```json
{ "code": "INVALID_MATH", "field": "<field>", "span": "<span>", "message": "<field>: <reason> near <span>" }
```

`message` repeats field and span because the SME portal's proxy forwards only `message`. Only the fields
an edit CHANGES are validated (ruling 20).

---

## 5. Version floor (old-client fallback)

```
MATH_MIN_VERSION = { android: '2.3.0', ios: '2.3.0', web: '2.3.0' }
```

A learner-facing response is rewritten with `mathTextToPlain` for an allowlisted set of text fields when
the client is below the floor for its platform. The platform is `X-Platform`
(`android` | `ios` | `web`); a missing or unrecognised `X-Platform` is judged against the **android**
floor (ruling 15). The version is `X-App-Version`, compared numerically per part: `2.3`, `v2.3.0`,
`2.3.0 (31)`, `2.3.0+31` all read as 2.3.0; anything not starting with a number is 0.0.0.
**A missing `X-App-Version` means old** (fallback applied). The fallback is OPT-IN per learner-facing
controller (`MathFallbackInterceptor`, ruling 18); keyed `/sme/*`, legacy `/cms/*`, admin and script
routes are never rewritten. Rewritten keys (string values, or arrays of strings, at any depth):
`question optionA optionB optionC optionD optionE answerExplanation solveTip approach explanation
answer modelAnswer questionText content text`. Fallback-enabled routes send
`Vary: X-App-Version, X-Platform`. The fast path (§1) guarantees maths-free bodies are returned
untouched (the same object).

---

## 6. Known hazards in live data and authoring rules (2026-11-11)

Recorded here so every implementation agrees on what these inputs do under v1 (each is a vector):

- **Coding-symbol options.** A coding-decoding option set uses `$` as a symbol: `C%B$F$E`, `B$F$E%C`,
  `E%B$F$C`. Under §2.3 `$F$` is inline maths, so plain text becomes `C%BFE` and the fallback would change
  the option's meaning. **Ruling 22:** these rows are fixed by a guarded SQL pass that escapes `$` as `\$`
  BEFORE go-live (the owner approves each statement at the time); the vectors keep the pre-fix behaviour on
  purpose. The prod text audit (B1b) lists every such row.
- **Markdown-escaped brackets.** A legal-text stem writes `\[the Armed Forces of the Union\]` mid-sentence.
  Resolved by ruling 21 (rule 2 is line-anchored): it is plain text.
- **Authoring rule — write currency as `\$`.** Pandoc-strict pairing means a currency `$` followed later on
  the same line by maths pairs the wrong dollars: `Price $5 and area $x^2$` is one inline span
  `5 and area $x^2`. SMEs (and prompts) must write `Price \$5 and area $x^2$`.
- **`\ce{…}` / `\pu{…}`** (chemistry) are allowed: the engine has mhchem on (ruling 23), so stage 2 skips
  their arguments (§4) and the measured suite lists both. (Found on 2026-11-11 as "fails stage 2"; resolved the same day.)
