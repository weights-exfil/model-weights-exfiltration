# How leaky is your inference API?

Detecting model-weight steganographic exfiltration from an LLM inference API — and the covert channel it hides in.

- **Read it:** https://weights-exfil.github.io/model-weights-exfiltration/
- **Try the demo:** https://huggingface.co/spaces/weights-exfil/inference-verifier-demo
- **Paper:** *Anytime-valid detection of LLM weight exfiltration*, arXiv:2610.11843

The page (Jekyll, served from `docs/`) walks through the threat model, how bits ride in a model's token
choices, and how inference verification with an anytime-valid (e-process) aggregate detector catches it.
The interactive demo — CPU-only, a recorded replay of the real model and detector — is embedded from the
Hugging Face Space above.

## Repository layout

- `docs/` — the published page (Jekyll). Figures are theme-aware inline SVGs under `docs/_includes/`.
- `LICENSE`, `NOTICE` — Apache-2.0 and attribution.
- `CITATION.cff` — how to cite (GitHub renders a "Cite this repository" button).

## Cite

See `CITATION.cff`, or cite the companion paper directly: *Anytime-valid detection of LLM weight
exfiltration* (arXiv:2610.11843).

## License

Apache-2.0 — see `LICENSE` and `NOTICE`. Only our own code and content are included; the upstream
`verifier/` is depended on, not redistributed here.

Authors: Mateusz Kowalczyk (NASK National Research Institute; MATS Research), Inés Ortega-Fernández
(MATS Research), Keri Warr (Anthropic).
