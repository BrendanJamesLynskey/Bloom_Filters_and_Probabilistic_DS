# Bloom Filters and Probabilistic Data Structures — Presentation

An interactive slide deck covering Bloom filters, HyperLogLog, Count-Min Sketch, cuckoo filters, MinHash, skip lists, reservoir sampling, and how to choose the right probabilistic structure. Aimed at mid-level software engineers.

## ▶ [Open Presentation](https://brendanjameslynskey.github.io/Bloom_Filters_and_Probabilistic_DS/index.html)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic |
|---|-------|
| 01 | Title slide |
| 02 | Why probabilistic data structures? |
| 03 | Bloom filter — core idea |
| 04 | Bloom filter — insert and query |
| 05 | False positive probability analysis |
| 06 | Optimal number of hash functions |
| 07 | Counting Bloom filters |
| 08 | Cuckoo filters |
| 09 | Quotient filters |
| 10 | HyperLogLog — cardinality estimation |
| 11 | HyperLogLog — how it works |
| 12 | Count-Min Sketch |
| 13 | MinHash — Jaccard similarity |
| 14 | Skip lists |
| 15 | Reservoir sampling |
| 16 | Comparison table |
| 17 | Applications — databases and caches |
| 18 | Applications — networks and analytics |
| 19 | Design trade-offs |
| 20 | Summary and further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | append `?print-pdf` to URL |

---

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) (Monokai) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm.

---

## References

- Bloom, B.H. "Space/Time Trade-offs in Hash Coding with Allowable Errors." *CACM*, 1970
- Flajolet, P. et al. "HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm." 2007
- Fan, B. et al. "Cuckoo Filter: Practically Better Than Bloom." CoNEXT, 2014
- Cormode, G. & Muthukrishnan, S. "An Improved Data Stream Summary: The Count-Min Sketch." 2005
- Mitzenmacher, M. & Upfal, E. *Probability and Computing*. Cambridge University Press
- [Redis Documentation](https://redis.io/docs/) — HyperLogLog, Bloom filter module, sorted sets

## License

Educational use. Code examples provided as-is. Standards references are to publicly available documentation.
