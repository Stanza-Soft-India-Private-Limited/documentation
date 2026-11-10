# ratex-support-cases-0.1.14.json — provenance

`ratex-support-cases-0.1.14.json` is a verbatim copy (sha256
`b3dc4a8dc13eafe140c70d4aa3a622439dbb319cf6d5cc995fad981c15af7b72`) of the `supportCases` array embedded in
RaTeX's public support table (`https://erweixin.github.io/RaTeX/demo/support-table.html`, fetched 2026-11-11
during the mobile engine spike M0): 1075 `{index, formula, score}` rows of RaTeX's main golden suite, each
rendered by RaTeX and by KaTeX 0.16.45 and scored by ink similarity (`ratex-ink-v2`, 1.0 = identical; mean
0.902; one row, `\includegraphics`, has no score). RaTeX-CMP 0.1.14, the mobile engine chosen by owner ruling
23, wraps that upstream RaTeX (KaTeX syntax, mhchem on). The upstream README's documented exclusions
(`\includegraphics`; `\htmlClass` `\htmlData` `\htmlId` not equivalent; `\htmlStyle` partial; prooftree
`\InsertBetweenHyps` `\ScoreTree` `\Cell` `\noCell`) are applied on top of the scores. The file is the input of
`scripts/math/build-allowlist.ts`, which generates `src/common/math/math-allowlist.ts` — the validation stage-3
allowlist of `docs/contracts/math-text-v1.md` §4 (ruling 17). Do not edit it by hand; replace it wholesale when
the engine version changes and re-run the script.
