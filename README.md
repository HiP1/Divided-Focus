# Divided Focus

**Separated Memory Spaces and Default-Deny Context Triage for LLM Context Management**

Ivan Phan (HiP) · ORCID: [0009-0003-1095-5855](https://orcid.org/0009-0003-1095-5855)

Native Memory Paper 2 · DOI: [10.5281/zenodo.19365086](https://doi.org/10.5281/zenodo.19365086)

## Summary

The context window of a large language model has no type system and no architectural separation between commands and data. Prompt injection vulnerability and context degradation over multi-turn conversations are treated as separate problems requiring separate solutions. This paper argues that the same architectural feature contributes to both, and that addressing it produces benefits for both.

The security argument is grounded in independent cross-lab evidence. A joint study by researchers across OpenAI, Anthropic, and Google DeepMind (Nasr et al., 2025) tested 12 published defences against adaptive attacks and bypassed all of them at above 90% success rates. The instruction hierarchy approach (Wallace et al., 2024, ICLR 2025) was among the defence classes that collapsed under adaptive pressure. The architecture and security arguments do not depend on the author's own preliminary findings in the Confidence Curriculum series, which are presented as corroborating observations consistent with the independent evidence.

The proposed architecture separates context into three memory tiers: L1 (command space, the only tier with directive authority), L2 (active data), and L3 (reference storage, garbage-collected). In the security-first mode, all incoming content enters L3 by default. Promotion to L1 requires passing two gates: trustworthiness evaluation under task-frame shift, then summarisation. The evaluation gate has independent peer-reviewed validation (Koulakos et al., 2024, BlackboxNLP at EMNLP). Platform triage serves as the performance-optimised alternative for trusted deployment contexts.

The paper identifies a verification inversion: in unmanaged context, verification and conversation quality compete for the same irrecoverable resource, creating a structural condition where fact-checking degrades quality. Managed context breaks this inversion. The paper also reframes the entanglement problem (understanding and instruction-following fused in the same weights) from weight-level disentanglement to contextual mode-setting and training-time separation, grounded in the observation that entanglement was never designed but emerged from training (Ouyang et al., 2022; Zverev et al., 2024). Empirical precedents (DRIP, ASIDE) demonstrate partial feasibility of de-instructionalising the data path without degrading comprehension. If de-instructionalised pathways are achieved, the two-gate promotion pipeline becomes structurally secure rather than probabilistically secure.

This is an architectural proposal and design argument, not an implementation report.

## Contributions

1. A unified architectural proposal arguing command/data memory separation, default-deny triage with two-gate promotion, tiered data retention, and post-inference context management from a single security requirement.
2. The concept of default-deny triage as a pre-inference security mechanism for LLM context.
3. Architectural requirements specifying that data-space content must be processed through pathways lacking instruction-following capacity.
4. The identification of a verification inversion where unmanaged context structurally penalises fact-checking, making context management a prerequisite for default verification behaviour.
5. A reframe of the entanglement problem from weight-level disentanglement to contextual mode-setting and training-time separation, with empirical precedents and the consequence that de-instructionalised data pathways make the gates structurally secure.

## Series

This is Paper 2 in the Native Memory series.

- **Paper 1:** [Strategic Forgetting](https://doi.org/10.5281/zenodo.19212126). Tiered persistence protocol for LLM context window management at the orchestration layer.
- **Paper 2:** Divided Focus (this paper). Separated memory spaces and default-deny triage at the native architecture level.

The paper builds on findings from the [Confidence Curriculum](https://hip1.github.io/confidence-curriculum/) series (Papers 1-5, Epilogue, Intro).

## Files

- `divided-focus.md` : Source manuscript (Markdown)
- `divided-focus.html` : Self-contained HTML with floating TOC, light/dark mode, inline SVG figures
- `divided-focus.pdf` : PDF render

## Citation

```bibtex
@misc{phan2026dividedfocus,
  author       = {Phan, Ivan},
  title        = {Divided Focus: Separated Memory Spaces and Default-Deny Context Triage for LLM Context Management},
  year         = {2026},
  month        = {3},
  doi          = {10.5281/zenodo.19365086},
  url          = {https://doi.org/10.5281/zenodo.19365086},
  note         = {Native Memory Paper 2. Pre-print.}
}
```

## Methodology

This paper was produced using a triad methodology. Three frontier models (Claude Opus 4.6, ChatGPT 5.4 Thinking, Gemini 3.1 Pro) served as adversarial reviewers across multiple review rounds. The author maintained sole editorial authority over all content and decisions. An observation from this process is documented in Section 12: three models assessed the entanglement problem as potentially intractable under a critical evaluation frame, then produced convergent implementation proposals under a constructive design frame after the problem was reframed. The observation is presented as hypothesis-generating, not evidential, and is consistent with the paper's proposal about contextual processing modes.

## Licence

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
