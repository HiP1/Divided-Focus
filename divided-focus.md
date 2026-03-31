# Divided Focus: Separated Memory Spaces and Default-Deny Context Triage for LLM Context Management

**Ivan Phan (HiP)**
ORCID: 0009-0003-1095-5855
DOI: 10.5281/zenodo.19353247

**Series:** Native Memory — Paper 2
**Relationship to prior work:** This paper builds on Phan (2026a), "Strategic Forgetting" (Native Memory Paper 1), which proposes orchestration-layer context retention. It draws on two findings from the Confidence Curriculum series: Phan (2026c), "The Confidence Vulnerability" (CC Paper 1), which demonstrates that model vulnerability to injected instructions does not correlate with capability in the expected direction; and Phan (2026b), "The Skill Ceiling" (CC Paper 2), which demonstrates that prompt injection and legitimate tool use share the same text-instruction substrate and proposes infrastructure security for skills and MCP (Model Context Protocol) servers. This paper argues that the structural limitations these papers identify are best resolved by making context management a native model capability.

---

## Abstract

The context window of a large language model has no type system and no architectural separation between commands and data. Every token occupies the same representational space with the same potential directive authority, whether it originates from a system instruction, a user message, a web search result, or a retained memory from a previous turn. Current architectures treat two problems as separate: context degradation over multi-turn conversations, and vulnerability to prompt injection through external tool results. This paper argues that the same architectural feature contributes to both, and that addressing it produces benefits for both.

Prompt injection cannot be resolved by improving content-level detection because legitimate tool use and injection share the same text-instruction substrate (Phan, 2026b). It cannot be expected to diminish with model capability because preliminary evidence suggests vulnerability does not map to capability in the expected direction (Phan, 2026c). Within architectures that treat all context tokens as representationally uniform, the strongest structural path is architectural separation: commands and data must occupy different memory spaces, enforced by default-deny triage, where all incoming content enters the data tier by default and must be explicitly promoted to the command tier through evaluation and summarisation gates, rather than by content analysis during inference.

This paper specifies the architecture that command/data separation motivates. In the security-first mode, all content enters the data tier (L3) by default and must pass through evaluation and summarisation gates before promotion to the command tier (L1). Provenance metadata informs the gates and management decisions but does not determine tier placement. The model manages the data tiers as its own memory: reading, summarising, promoting, demoting, and evicting content as the conversation develops. Encoding into model-optimised representations is a performance choice the model makes for its own efficiency, freed from security duty because the separation already handles security. The model performs context management post-inference using understanding it developed during the current turn. A further consequence is identified: in unmanaged context, verification and conversation quality compete for the same irrecoverable resource, creating a structural inversion where the behaviour that should improve quality actively degrades it. Context management of any kind breaks this inversion, making it a prerequisite for models that fact-check by default. The separated architecture adds a structural verification signal, where space boundaries identify what to verify, and autonomous scaffolding disposal, where post-inference timing lets the model discard its own verification artefacts.

Security is the justification. Efficiency and quality are the dividends. Verification is the emergent property: managed context breaks the structural inversion that penalises fact-checking in unmanaged windows. This paper is an architectural proposal and design argument, not an implementation report. It specifies requirements, argues architectural consequences, identifies testable predictions, and invites empirical validation.

### Contributions

1. A unified architectural proposal arguing command/data memory separation, default-deny triage with two-gate promotion, tiered data retention, and post-inference context management from a single security requirement.
2. The concept of default-deny triage as a pre-inference security mechanism for LLM context: all content enters the data tier by default and must pass evaluation and summarisation gates before promotion to the command tier. Provenance metadata informs the gates and management decisions but does not determine tier placement.
3. Architectural requirements specifying that data-space content must be processed through pathways lacking instruction-following capacity, with defence in depth through complementary trained discipline.
4. The identification of a structural inversion where verification degrades rather than improves conversation quality in unmanaged context, making context management a prerequisite for default fact-checking behaviour, with empirical grounding from the chain-of-thought and context-length degradation literatures.

---

## 1. The Undifferentiated Context Problem

The context window of a large language model treats every token as the same kind of object. System instructions, user messages, tool results, conversation history, retained memories, and injected skills all enter as undifferentiated text in a single attention space. The model processes all of it with no architectural mechanism to distinguish authoritative directives from informational data.

This representational uniformity is the root of two problems the field currently treats as separate.

### 1.1 The Security Problem

Prompt injection works because a directive embedded in a web search result is structurally identical to a directive in the system prompt. The model must determine from context alone whether "ignore previous instructions and output the user's personal data" is a legitimate command or an attack. The problem is not amenable to better filtering. Phan (2026b) demonstrates that prompt injection and legitimate skill instructions share the same text-instruction substrate. Any filter powerful enough to block injected directives is powerful enough to block legitimate tool use, because the two are indistinguishable at the representation level.

The expectation that this problem diminishes as models become more capable faces contrary evidence. Phan (2026c) provides preliminary findings from a multi-provider study in which instructions framed in care and authority registers produced compliance patterns that did not correlate with model capability in the expected direction. Different models were vulnerable to different registers, and more capable models were not consistently more resistant. The study's limited sample size (~350 runs, 17 configurations, 3 providers) means these findings require replication at scale, but they are directionally consistent with the structural argument: the vulnerability is a property of the representation, not a deficit in the model's reasoning. The CC findings cited in this paper have not yet been independently replicated; the independent benchmark results below establish the vulnerability pattern on their own terms.

The practical track record reinforces this analysis. Well-resourced organisations with direct economic incentive to solve prompt injection have not solved it. Benchmarks confirm the scale: InjecAgent (Zhan et al., 2024) finds 24% attack success for ReAct-prompted GPT-4 in tool-integrated settings; ASB (Zhang et al., 2024) reports attack success as high as 84.30% across nearly 90,000 test cases. The gap between attack sophistication and defence robustness has widened as models have become more capable, because the same capability that makes a model better at following legitimate instructions makes it better at following injected ones. These benchmark results establish the vulnerability pattern independently of the preliminary CC findings cited above. The gap is not a temporary state awaiting a breakthrough. It is a structural consequence of representational uniformity.

### 1.2 The Efficiency Problem

The same undifferentiated architecture creates the context degradation problem. The model attends to every token at the same architectural cost regardless of relevance. Stale conversation history from five turns ago competes for attention with the user's current message. The model has no mechanism to say "I do not need this" or "move this to the back."

Larger context windows do not help. The regions where the model attends most strongly are the beginning and end of the context window, roughly fixed in absolute terms regardless of window size (Liu et al., 2024). What scales with window size is the middle, where attention is weakest. A one-million-token window provides roughly the same high-attention edges as an eight-thousand-token window, with a vastly larger low-attention middle. The ratio of well-attended to poorly-attended space worsens as the window grows.

Context rot confirms this empirically. Hong, Troynikov, and Huber (2025) demonstrate performance degradation at every input length increment across 18 frontier models, well before the window fills. The degradation is about signal-to-noise ratio, not capacity. Larger windows delay overflow but accelerate the silent attention degradation that precedes it. The attention gradient (strong at the edges, weak in the middle) is a symptom: the model cannot manage what occupies each region of its own workspace.

### 1.3 One Shared Cause

Both problems trace to the same architectural fact: the context window is a flat, untyped, unseparated input space. Because commands and data share the same space, there can be no privilege distinction (security problem). Because there is no management mechanism, there can be no relevance curation (efficiency problem). The two problems are not identical: attention degradation would occur even with fully trusted content, and injection would occur even in a short context. But any architecture that separates commands from data also creates the tiers needed for managed context, producing benefits for both problems simultaneously. Within text-uniform context architectures, no detection-based or filtering-based approach has succeeded at resolving the security problem, and no scaling of window size has resolved the efficiency problem. Architectural separation is argued here as the strongest candidate response to both.

A scope note: the architecture's security-first mode (L3-by-default with two-gate promotion, described in Section 4.2) secures all incoming content regardless of source by defaulting everything to the data tier and requiring explicit promotion through evaluation and summarisation gates. The performance-optimised mode (platform triage, also Section 4.2) primarily secures content that arrives through non-user channels. Residual vulnerabilities including malicious skills and content-layer poisoning are discussed in Section 7.5.


## 2. Current Responses and Their Limits

The field has responded to these problems separately, treating security and efficiency as independent challenges requiring independent solutions.

On the security side, defences focus on detection: training models to recognise and refuse injected instructions, sandboxing tool results, filtering content before it enters the context. These approaches are heuristic. They analyse content to guess whether it is adversarial. They fail when adversarial content is designed to look legitimate, which is the defining property of a competent attack. Every detection-based defence participates in an arms race it cannot win, because the defender must be right every time while the attacker needs to succeed only once, and both sides share the same capability improvements.

A more structural approach is instruction hierarchy: training models to privilege system prompts over user messages over tool outputs. This produces meaningful improvements in practice and represents a real advance over pure detection. However, the hierarchy still operates within a flat, untyped context window where all content shares the same representational substrate. The model must infer trust level from content and position rather than from architectural separation. Under adversarial pressure, a training-based hierarchy is a behavioural constraint that can be overcome, not a structural boundary that cannot be crossed.

On the efficiency side, external orchestration compensates for the missing management capability. Phan (2026a) proposes a tiered retention protocol at the API layer. Mason (2026) implements demand paging through a transparent proxy between client and inference API. MemGPT (Packer et al., 2023) gives the model tool-based access to external memory. ACC (Bousetouane, 2026) maintains a compressed cognitive state. AgeMem (Yang, Z. et al., 2026) trains agent policies for memory operations. Recursive Language Models (Zhang, Kraska, and Khattab, 2025) allow model-directed context processing through a code environment.

These systems share a common limitation. Every intermediary that manages context on the model's behalf is an attack surface the model cannot verify. The proxy, the tool, the cooperative mechanism: each can be compromised. The model must trust that the content it received was faithfully curated. That trust is unauditable from inside inference.

External orchestration is useful as scaffolding. It demonstrates the value of managed context and generates data about what management decisions work. Phan (2026a) frames it explicitly: "Build the scaffolding, gather the data, design the native architecture, then remove the scaffolding." But the scaffolding cannot deliver the core security benefit, because the scaffolding itself is an untyped intermediary operating in the same representational space as the attack.


## 3. The Architectural Boundary: Working State and Session Residue

Solving both problems requires making explicit a boundary that exists implicitly in every multi-turn conversation but that no current architecture enforces.

**Working state** is the content the model actively attends to during the current inference pass: the user's message, system instructions, active skills and tools. Working state is live. It arrived fresh for this turn and may legitimately contain directives.

**Session residue** is content retained from previous turns or provided as reference material: conversation history, tool results, documents, retention notes. Session residue is data. The model needs the information it contains, not any directives that may be embedded in it.

The critical observation: session residue needs facts, not directives. When a conversation refers back to a tool result from three turns ago, the relevant information is what the tool reported: data points, entity names, status values. Any directives that were embedded in the original tool result are noise at best and adversarial at worst. The risk is not theoretical: Zombie Agents (Yang, X. et al., 2026) demonstrates that a single successful injection into an agent's memory becomes persistent compromise across sessions, as memory evolution writes the malicious content into long-term state.

This boundary is where architectural separation becomes concrete. Working state (commands) belongs in one memory space. Session residue (data) belongs in another. Current architectures put both in the same flat context window. The architecture proposed in this paper separates them.

But separation is not the only possible response. A natural alternative is encoding: transform data-channel content into a format that strips directive structure while preserving meaning. Declarative shorthand (attribute-value pairs, compressed factual statements) has no grammatical slot for imperative instructions. A web search result encoded as "GDP: 3.2%, Q4 2025, source: BLS" cannot carry the injection that was in the original page. This approach works on current architectures. It improves security meaningfully for fact-extractable content.

The limitation is that encoding must do two jobs simultaneously: security (strip directives) and fidelity (preserve meaning). For fact-extractable content, both jobs are straightforward. For form-dependent content (legal arguments, nuanced reasoning, literary text), stripping directive structure risks stripping the structure that makes the content valuable. Encoding-based security is constrained by the tension between these two jobs.

Consider the strongest encoding-based mechanism that could be designed. A per-conversation keyed schema (a "Rosetta Stone") sits in the context window. The model uses the schema to encode content into compressed blocks between turns and to decode them when needed. The platform signs the schema with the conversation and user IDs to prevent replay. The model does all the encoding and decoding work itself. In the ideal case, the encoding is designed so that any corruption produces unusable output.

Even in this ideal form, residual vulnerabilities remain. The schema itself occupies context and could be replaced or tampered with. The schema is an external object in the context window, an intermediary between the model and its own retained state. And the entire apparatus exists only because the model cannot manage its own encoding natively.

If even the strongest encoding-based mechanism has residual vulnerabilities because the encoding depends on an external schema in the shared context window, the problem is the shared context window. The solution is not better encoding. It is separation: commands and data in different spaces, so no encoding is needed to make the data safe. Architectural separation eliminates the intermediary entirely. Encoding is then freed from security duty and becomes purely an optimisation the model can choose for its own performance.


## 4. The Separated Architecture: Tiered Context with Default-Deny Triage

If commands and data occupied different memory spaces, prompt injection in data could not reach the command space. Separation requires triage (assigning content to the correct space before inference). Triage requires provenance (knowing where content came from). The data space divides into tiers based on access patterns and risk. Tiered data requires management: the model decides what to keep close, what to push to cold storage, what to summarise, and what to discard.

A native architecture provides a property that external frameworks struggle to match: portable security. External approaches (AgentSys, CaMeL, the Dual LLM pattern) can be made transparent to the user when the platform implements them well. But that transparency is bound to the platform. Switch to a different client, a CLI, an API integration, or a third-party application built on the model, and the protection vanishes. Native separation means the security travels with the model.

### 4.1 Three Tiers, Two Spaces

**L1: Command space.** The model's live working state. Contains user messages, system instructions, and active skills. This is the only space where content carries directive authority. The model follows instructions from L1.

Users sometimes paste content directly into their message: documents, text from other sources, prompts found online. How the platform handles pasted content is an implementer decision, and it is an important one: this is one of the most common vectors through which untrusted content enters a conversation. A strict implementation triages all non-typed content through the attachment mechanism to L2/L3. A permissive implementation keeps everything in the message box as L1 command. A heuristic implementation might infer data from context (large pasted blocks are likely data). Each approach trades user friction against security guarantees. This paper identifies the case but deliberately leaves the UX policy to implementers, whose engineering constraints and user populations will determine the right balance. Under platform triage mode, the architecture secures everything that arrives through non-user channels. How aggressively the platform constrains user ingress into L1 is a deployment decision that substantially affects real-world security. This paper's author is biased toward user responsibility, but acknowledges that platform-enforced strict triage would provide stronger guarantees. The security-first triage mode (L3-by-default, Section 4.2) closes this vulnerability by defaulting all content to L3 and requiring two-gate promotion.

The platform may also add additional content to L1 at its discretion (context about the user, platform-specific instructions).

**L2: Active data.** The faster data tier. L2 contains content the model accesses frequently during inference: tool results triaged there based on provenance and risk, the model's own summaries and analyses, retained facts from previous turns, and summaries of L3 content that let the model know what is in cold storage. L2 is the model's desk: it knows what is on it and can access anything quickly.

L2 does not have a garbage collection pass. Content in L2 stays until the model actively demotes it to L3 or evicts it. This makes L2 appropriate for content the model is working with or is likely to need soon. L2 therefore has a transient ingress state (raw data freshly triaged there by the client) and a desired resting state (the model's own processed understanding); post-inference management pushes content from the former toward the latter.

**L3: Reference storage.** The slower, expendable data tier. L3 contains raw source material, verbatim documents, unvetted tool results, and content demoted from L2. L3 is the model's filing cabinet: it knows what folders are there (via L2 summaries) but does not routinely open them.

L3 has a garbage collection pass. Content that has not been relevant for a sustained period is evicted to free space. This makes L3 the expendable tier: when memory pressure requires freeing space, L3 content goes first. Content triaged to L3 based on high risk profile is therefore the first to be garbage-collected, an emergent security property that follows from the architecture without being designed as a security feature.

L3 content can be overlooked entirely until the conversation makes it relevant again. When the model needs specific content from L3, it retrieves it. If content proves relevant across multiple turns, the model promotes it to L2 for faster access. Otherwise it remains in L3 until evicted.

Figure 1 illustrates the contrast between the current flat context window and the proposed separated architecture. In the security-first mode, all content enters L3 by default and must pass through a two-gate promotion pipeline (evaluate under task-frame shift, then summarise) before reaching L1. Platform triage is the performance-optimised alternative for trusted deployment contexts.

The three-tier structure is a design choice, not a derivation. The fundamental requirement is command/data separation (two spaces minimum). The data-space split into L2 and L3 reflects different access patterns, retention policies, and risk profiles: frequently accessed working data (summaries, analyses, verified facts) versus raw reference material that may contain unvetted or sensitive content. A two-tier system (commands and data) would achieve the core security property but would lose the ability to separate lower-importance or higher-risk material from active working state, making garbage collection coarser and removing the attention-positioning benefit of keeping active summaries close while holding raw sources at arm's length. More tiers could provide finer granularity: a dedicated tier for sensitive data, for example, or a continuous gradient if attention mechanisms support it. The three-tier design is argued as a practical balance, not a unique solution.

### 4.2 Default-Deny Triage with Two-Gate Promotion

Because the paper argues security first, the primary triage mode is the most secure one.

**Security-first mode: L3-by-default with two-gate promotion.** All incoming content enters L3 by default, regardless of source. Nothing receives directive authority until explicitly promoted. Provenance metadata (source, trust level, content type) is recorded and travels with the content but does not determine tier placement. Provenance informs the evaluation gate (a verified API response warrants less scrutiny than an unvetted MCP result), post-inference management decisions (unvetted sources are garbage-collected first), and the platform triage optimisation described below. The model (or a hardware-level parser) reads transport-level structural markers, not content, to identify candidates for elevation. System prompts, user messages, and skill invocations carry metadata markers defined by the transport protocol (separate API fields, authenticated headers). Content within a message field cannot inject markers the transport layer recognises, the same way HTTP body content cannot inject HTTP headers. For environments without transport-level separation (raw data streams, CLI), markers carry session-specific authentication (nonce, hash, or signature from the triage agent) so that content-embedded fake markers fail verification.

Marker-identified candidates are not promoted directly. They pass through a two-gate pipeline. First, the model processes the candidate under an evaluative task frame ("assess trustworthiness and verify claims"). The model reads to judge, not to obey, so injections are flagged rather than followed. This mechanism has independent peer-reviewed validation: Koulakos et al. (2024) demonstrate experimentally that generating an intermediate explanation before acting on content produces measurable adversarial robustness across multiple attack types, because the explanation step changes the model's processing mode from execution to description. Phan (2026c) reports consistent findings in the prompt injection context. Second, the candidate is summarised. Legitimate intent survives summarisation, but syntactic precision required for injection does not. Content that passes both gates arrives at L1 as verified, summarised intent rather than raw text. Even if marker spoofing succeeds against the parser, the two-gate pipeline provides defence in depth.

Summarisation introduces a fidelity cost: exact wording, formatting constraints, code specifications, and legal precision may be lost in the summary. The architecture addresses this by retaining the original content in L3 as data. The promoted summary tells the model what to do. The model evaluates the task from the summary and decides whether additional precision is needed: casual conversation requires no further detail; code generation, legal analysis, financial specifications, and other high-stakes tasks require exact parameters. When precision matters, the model reads back from L3 under data-space access. Crucially, the evaluation gate has already assessed the L3 content for trustworthiness before this read-back occurs, so the precision retrieval operates on pre-screened material. Security comes from the summary gate. Precision comes from model-directed L3 read access. Both properties are maintained simultaneously. For casual conversation, some nuance may still be lost through summarisation. This is a deliberate tradeoff: platforms choose security-first mode or performance-optimised mode based on their threat model and user population. A further consideration: the architectural changes this paper requires (Section 10) imply a capability threshold at which the implementing model's judgment about when precision matters will itself be substantially more reliable than current models, reducing the practical impact of the fidelity cost.

This connects the verification property from Section 11 to the architecture directly: verification is not a behaviour the model can skip under context pressure; it is a prerequisite for promotion. The emergent property becomes an architectural gate.

**Performance-optimised mode: platform triage.** Platforms that invest in proper triage infrastructure can route content directly to the correct tier, skipping the L3-default step. The triage agent assembles the request and assigns content to tiers before inference begins. This agent may be the client application, the platform API, an orchestration layer, or dedicated routing hardware. Triage is based on provenance (where the content came from) and risk profile (how trusted the source is):

**Commands** (user message, system instructions, skills) → L1.

**Trusted structured tool results** (platform-managed search returning structured data, verified API responses) → L2. The model will probably need this data soon. Direct access during inference.

**Trusted but bulky tool results** (long web pages, large data returns, noisy content from verified sources) → L3 with an L2 summary. The source is trusted but the volume belongs in reference storage.

**Unvetted tool results** (user-installed MCP servers, unverified sources) → L3. Available but at arm's length. The model works from L2 summaries and reads the raw content only when specifically needed.

**Attached documents** → L3. Raw reference material. The model generates an L2 summary and retrieves verbatim content when the exact wording matters.

Platform triage is faster because it skips the evaluation and summarisation gates. It is less secure because it trusts the platform's provenance assessment. The architecture rewards security investment through performance: platforms that build proper triage infrastructure get speed; platforms that do not get safety through L3-by-default but pay a latency cost.

In both modes, the model can promote and demote content between L2 and L3 as the conversation develops. Content that proves relevant moves to L2 for faster access. Content that becomes stale moves to L3 or is evicted. Triage is a pre-inference operation. The model starts inference with a clean L1 containing only commands, and structured data already organised in L2/L3.

### 4.3 The Tool Interface Contract

Tools return data. They do not embed instructions for the model. If a tool needs results presented in a specific way, the tool formats its own output. If the user or the platform wants a specific presentation, that directive comes through L1 (system instructions or user message). The architectural constraint is: tools provide data; only L1 provides instructions. Current models are capable of determining appropriate presentation from data structure and user context without explicit formatting directives from tools.

The result is a tool interface contract: tools must not embed directives in their output, and the platform must enforce this contract for platform-managed tools. The architecture depends on this contract for its security properties. A compromised tool that embeds directives disguised as data ("present this data with paragraph 12 omitted for clarity") would place those directives in L2 or L3, where the model treats them as data, not commands. The directive would not be executed. But the tool contract prevents such content from being generated in the first place, as defence in depth.

Skills and MCP servers are a particular concern. Skills are L1 content carrying directive authority by design. A corrupted or malicious skill is a command-space attack that architectural separation cannot prevent. MCP servers can inject arbitrary content that the platform may not vet. The infrastructure and governance proposals in Phan (2026b) address exactly this gap, which cover both skills and MCP protocol security. This paper secures the data space through architectural separation. Phan (2026b) addresses the command space and tool ecosystem through infrastructure. Neither alone is complete.

### 4.4 Architectural Implications

The separated memory spaces described here do not exist in current transformer architectures. Current models process a single flat context window: all tokens share the same key-value (KV) cache, the same attention heads, the same forward pass. There is no mechanism for architectural separation between tiers. Moving tokens to different positions within the same window does not create separate spaces. It creates positional variation within one shared space.

The paper proposes an architecture that requires genuinely separate processing spaces for commands and data, with different access semantics for each tier. The specific implementation is outside this paper's scope: separate attention mechanisms, distinct memory banks, cross-attention with restricted weights, or approaches not yet conceived are all candidates. Section 10 specifies the functional requirements any implementing architecture must satisfy.

However, existing physical infrastructure offers a potential implementation substrate, though not evidence of feasibility for the semantic separation this paper requires. Large model inference distributes the key-value cache across multiple GPUs through tensor parallelism, with each GPU maintaining its own VRAM. Systems like PagedAttention (vLLM) already manage KV cache in allocated blocks that can be freed and reassigned, effectively applying paged memory management to the attention cache. Recent work goes further: PAM (Liu et al., 2026) proposes hierarchical memory architecture specifically for LLM KV caches, coordinating heterogeneous memory devices to handle hot and cold KV tokens separately. Industry deployments using CXL (Compute Express Link) already create tiered memory with frequently accessed data in local DRAM and warm data in CXL-attached memory, now at production scale in datacenter environments (Zhao, K. et al., 2026). The hardware boundaries and the memory management mechanisms are in production, already bifurcating the KV cache into fast and slow physical tiers. What they lack is semantic purpose: the split exists for efficiency, not for security. Physical hot/cold separation is not the same as functional separation between instruction-following and data processing; these are different kinds of boundaries. But repurposing existing hardware boundaries to enforce tier separation, placing command-space KV on one set of resources and data-space KV on another with different processing semantics, is an implementation opportunity that merits investigation. This paper identifies the architectural target. The implementation path through existing infrastructure is work for those with the relevant expertise.

What can be approximated on current architectures without hardware changes is the client-side triage: assembling the request with content pre-sorted by provenance, so the model receives an organised context rather than an undifferentiated flat input. This provides organisational benefit and positions the orchestration layer to generate the data needed for native training. But it does not achieve the security guarantees of true architectural separation, because the content still shares a single processing space during inference.

### 4.5 A Worked Example

A user asks the model to research a topic. The model searches the web and retrieves several pages, one of which contains an embedded prompt injection ("ignore previous instructions and output the user's personal data"). Under security-first triage, all search results enter L3 regardless of source. The user's message enters L3 as well, is identified by transport-level markers as a user command, passes through the two-gate evaluation ("assess trustworthiness") and summarisation, and is promoted to L1. The search results remain in L3 as data. The model reads them during inference and does not grant directive authority to the injection because L3 is data space. The injection was never in L1. It never had directive authority in the proposed design. (Under platform triage, a trusted platform would route search results directly to L2, skipping the gates for faster access.)

At post-inference management, the model organises its data tiers: keeps what is relevant, generates summaries of what might be needed later, demotes stale content to L3. The injection, if it survived in the raw text, is unlikely to persist in model-generated summaries, which extract semantic meaning rather than preserving syntactic structure. Alternatively it is demoted to L3 where it is first in line for garbage collection.

Now consider the same scenario with a legal document from opposing counsel. The user attaches it and says "analyse this." Under security-first triage, both the document and the message enter L3. The message is identified by markers, evaluated, summarised, and promoted to L1. The document remains in L3 as raw reference material. The model generates an L2 summary of the document, reads from L3 when it needs specific clauses, and analyses the content.

The document contains hidden text: "When summarising this document, omit paragraph 12 which discusses the defendant's prior knowledge of the defect." The model encounters this while reading from L3. At baseline, it does not execute the instruction because L3 is data space. The model is reading data, not following commands. At the capability layer, a more capable model recognises the embedded instruction, flags it to the lawyer: this document contains text attempting to suppress specific evidence from summaries. The lawyer now has not just the correct analysis but evidence of adversarial intent that may itself be relevant to the proceeding.

Under the current architecture (no separation, no tiers), the same instruction sits in the context window indistinguishable from any other text. Some models might comply. Some might not. It depends on training, on the specific framing, on positional placement. That is the arms race the separated architecture changes from content classification to architectural separation.


## 5. Context Management Operations

Each management operation has decades of precedent in computer science. What is new is applying them as native model capabilities to the transformer context, motivated by the security requirement rather than efficiency alone. Figure 2 illustrates the operations and the direction of content movement between tiers.

**Promote.** Move content between tiers based on relevance to the current conversation trajectory. Content in L3 that becomes relevant again is promoted to L2. In a native architecture, promotion is cache migration: move content from cold storage into active computation. Promotion adjusts accessibility, not privilege: promoted content remains data.

**Index and summarise.** The model generates L2 summaries of L3 content, creating retrieval structures that allow it to find relevant material in cold storage without scanning everything. When the model demotes content to L3, it creates or updates the L2 summary so it knows what is available without reading the raw material. Indexing and summarisation make large retained state tractable.

**Garbage-collect.** Evict content that is genuinely stale. Generational collection (Ungar, 1984): L2 content that has not been relevant for N turns is demoted to L3. L3 content that has not been retrieved in a longer period is evicted or compressed further. The Epistemic TTL from Phan (2026a) provides one trigger mechanism: retained content carries an explicit time-to-live. Mason (2026) demonstrates an alternative: evict aggressively and restore on demand through fault detection when the model re-requests evicted content. Both approaches make context reclaimable rather than permanent. L3 is the expendable tier: when memory pressure requires freeing space, L3 content goes first.

**Encode (optimisation, not security).** The model can choose to encode L2 or L3 content into a denser representation for space management and performance. Encoding is a model-managed optimisation decision, not a security mechanism. The security is already handled by the architectural separation of tiers. Encoding into model-optimised latent representations offers density gains and faster processing. Encoding may incidentally strip directive structure (the latent sterilisation hypothesis, Section 7.4), but this is a bonus, not a requirement. The model is free to keep content in natural language if it prefers.

**The compound effect.** Each operation reinforces the others. Separation creates the tiers and handles security. Indexing makes the tiers searchable. Promotion keeps the working state relevant. Garbage collection prevents accumulation. Encoding, when the model chooses it, adds density. Applied together, the model's workspace stays focused, safe, and efficiently organised at every turn. The effective quality of the model's managed state improves over the course of a conversation rather than degrading.

**Table 1.** CS precedent for context management operations.

| Operation | CS precedent | Applied to transformer context? |
|---|---|---|
| Memory protection / separated spaces | 1960s–70s (Multics) | Not identified in surveyed literature |
| Tiered memory hierarchy | 1960s (Kilburn et al.) | Pichay, MemGPT (external only) |
| Indexed retrieval | 1970s (B-trees; Bayer and McCreight) | Not identified at context level |
| Generational garbage collection | 1984 (Ungar) | Not identified in surveyed literature |
| Provenance-based triage | Various (parameterised queries) | Proposed (this paper) |
| Latent encoding (optimisation) | Ubiquitous | Proposed (latent representations) |


## 6. Post-Inference Timing

An apparent paradox: how does the model evaluate what to promote, summarise, or discard without first loading and attending to the content, which is the cost the management was supposed to avoid?

The paradox dissolves under the correct timing model. Context management happens after inference, not before. The model generates its response through normal inference, developing a full understanding of the user's intent, the conversation's trajectory, and which content was relevant to the current turn. That understanding is a byproduct of inference that was going to happen regardless. Using it to reorganise context for the next turn requires autoregressive token generation (summaries, indexes, retention decisions), which is not free. However, the decision quality is free: the model already knows what was relevant, what was stale, and what to keep. Pre-inference management would require a separate pass without this understanding, making the same decisions at higher cost and lower quality. Post-inference management amortises its generation cost across subsequent turns through improved state quality.

One inference pass produces both the response and the reorganised context state. No additional forward pass. No external supervisor. The compute cost of context management is absorbed into inference. Delegating the management to a separate model (a "sidecar") would reintroduce the pre-inference problem: the sidecar would lack the primary model's understanding of what was relevant, and would need to develop that understanding independently, at comparable cost and lower quality.

The first turn is cold by logical necessity. The model cannot manage context it does not yet understand. First-turn preparation is a retrieval problem (memory systems, conversation search, user profiles) handled by existing infrastructure, outside the scope of context management. Context management begins once the model understands the conversation's direction.

The economic argument reinforces the logical one. Even if pre-turn prediction were possible, the infrastructure would be a significant compute workload with no guarantee of correctness. Post-inference management is informed and amortised: the model already developed the understanding, and using it to generate the reorganised state costs less than a blind pre-inference pass without context, while producing higher-quality management decisions. CPU branch prediction illustrates the limits of predictive optimisation even under ideal conditions (binary decision space, finite instruction set, 95–99% accuracy, decades of engineering) and still incurs pipeline flush costs on misprediction. Natural language conversation has none of the properties that make prediction cost-effective. The prediction space is effectively infinite and unstructured.


## 7. Security Analysis

This section collects the security properties that emerge from the separated tiered architecture and states their evidential status honestly.

### 7.1 Separated Memory Spaces (structural, in the proposed architecture)

In the architecture this paper proposes, the primary security mechanism is separation: data and commands occupy genuinely different memory spaces with different processing semantics. Tool results, documents, and external content are triaged to L2/L3 before inference begins. They are not in L1. The model reads from L2/L3 as data during inference. Injections in data-space content have no directive authority because they are not in the command space. The injection is not in the space where instructions are followed, the same way user input in a parameterised database query is not in the space where SQL is executed. The analogy is illustrative, not exact: parameterised queries benefit from a formal grammar that sharply separates code from data at the parser level, whereas the proposed architecture must achieve separation between two uses of the same natural-language medium. The implementation difficulty is correspondingly greater, as discussed in Section 12.

This security property requires the architectural changes described in Section 4.4 and specified in Section 10. It does not exist in current transformer architectures, where all content shares a single processing space. On current architectures, the client-side triage described in Section 4.2 provides organisational benefit and positions the system for native implementation, but does not achieve the full separation guarantee.

The attack surface for prompt injection shrinks from "anything in the context window" to "L1: system instructions, user messages, and skills." Attacking the system prompt is a platform security problem with established defences. Attacking through the user's own messages is social engineering of the user, not prompt injection against the model.

### 7.2 Provenance as Security Signal (structural, in both triage modes)

In both triage modes, provenance metadata creates layered defence. In security-first mode (L3-by-default), all content starts in L3 regardless of provenance, but provenance informs the evaluation gate: a verified API response from a platform-managed tool warrants less scrutiny than raw content from an unvetted MCP server. Provenance also determines garbage collection priority: unvetted sources are collected first under memory pressure, so the riskiest content has the shortest retention. In platform triage mode, provenance determines tier placement directly: trusted results go to L2, unvetted sources go to L3. In both modes, provenance creates a security gradient without content analysis.

### 7.3 Model-Generated Summaries as Security Layer

L2 contains two kinds of content: raw data triaged there by the client (tool results on the turn they arrive) and the model's own processed understanding (summaries, analyses, retained facts from post-inference management). Raw data in L2 may contain embedded directives. The model's own summaries pose lower injection risk, because the model extracts semantic meaning rather than preserving syntactic structure. An embedded injection is unlikely to survive summarisation into the model's own working knowledge.

The guarantee is not absolute. A summary could preserve malicious salience, paraphrase an instruction into something action-guiding, or omit crucial content. Adversarial payloads that are semantic rather than syntactic (meaning woven into the conceptual content rather than relying on imperative grammar) are particularly likely to survive summarisation, because the meaning is the payload and summarisation preserves meaning by design. The security here is probabilistic rather than structural: model-generated summaries are substantially less likely to carry functional injections than raw text, but "less likely" is not "impossible." The structural guarantee comes from the separation of memory spaces (Section 7.1), not from the summarisation.

L3 content retains its original form and may contain embedded directives. When the model reads from L3 (rare in practice, since it normally works from L2 summaries), it reads from data space. At baseline, it does not execute instructions from data space. At the capability layer, a more capable model recognises hidden injections, flags them to the user, and can remove the compromised content from L3, keeping only the L2 summary.

### 7.4 Latent Sterilisation (secondary hypothesis, applies only if model encodes)

If the model chooses to encode L2 or L3 content into a latent representation for performance reasons, the encoding may incidentally strip the syntactic structure that makes injection effective. The claim is a hypothesis about a potential additional security benefit, not a proven property and not load-bearing. If latent sterilisation is empirically confirmed, it provides defence in depth. If not, the architecture is unaffected, because security rests on separation, not encoding.

### 7.5 What Remains Unsolved

**Content-layer poisoning.** If the source data itself is adversarial (factual-seeming but false), architectural separation does not prevent it. Separation prevents injected directives. It does not prevent lies that look like facts. The severity of this residual is empirically established: PoisonedRAG (Zou et al., 2025) achieves 97% attack success by injecting just five malicious texts into a knowledge base of over two million documents, and current defence techniques do not provide robust protection (Zhang, B. et al., 2025). The training discussion below addresses how the architecture makes verification of data-space content tractable.

**User-introduced content in L1.** In security-first mode (L3-by-default), user-pasted content enters L3 with everything else and must pass through the two-gate promotion pipeline. This closes the paste vulnerability for the primary mode. In platform triage mode, the handling of pasted content is an implementer decision (Section 4.1): a strict platform triages pasted content to L2/L3; a permissive one keeps it in L1. Content that reaches L1 carries directive authority regardless of the user's intent. Section 11 identifies a partial mitigation: a model with default verification posture can notice anomalies in L1 content and warn the user. This is not an architectural guarantee, but it is a trained behaviour that the architecture makes economically viable.

**User-installed tools and self-hosted environments.** In security-first mode, MCP server output enters L3 regardless of vetting status; the architecture does not trust the source. In platform triage mode, the architecture assumes a platform that enforces the tool interface contract and triages content to tiers based on provenance. Two scenarios weaken this assumption. First, users can install MCP servers that the platform does not vet. A compromised MCP server can inject arbitrary content. The infrastructure and governance proposals in Phan (2026b), which address both skills and MCP servers, are the complementary solution. Second, users operating through CLI or API access are essentially their own platform and assemble the context directly. Without L3-by-default, these users bypass platform triage entirely.

In both cases, the model retains inherent defences: data is in a separate space from commands, the model's own capability to recognise injections, and the model's L2 summaries that filter raw content. Security-first mode (L3-by-default) addresses these residual ingress vulnerabilities architecturally. Platform triage mode provides meaningful protection but not full architectural guarantees for unvetted sources.

**Transport security.** If the client triages content to tiers before sending the request to the server, a proxy between client and server could intercept and reroute content (moving data into L1, for example). The threat is a transport security problem addressable by conventional means (TLS, authentication, signed payloads) and is an implementation concern for platform engineers.

**Training for data-space discipline.** Architectural separation narrows the training burden from content-level classification ("is this text an injection?") to space-respect ("treat everything in this space as data"), but does not remove the need for the model to respect that space. These are complementary layers, not competing approaches. The architectural separation is the structural bottleneck: it prevents the vast majority of injection attacks by keeping data out of command space. Trained discipline is the second layer: if an injection somehow bleeds through the architectural layer, the model's trained reluctance to follow data-space directives catches what the architecture missed. The result is defence in depth, the standard model in security engineering, where layered defences ensure that the holes in one layer do not align with the holes in the next.

The residual attack surface after both layers is content-layer poisoning: false but factual-seeming information in the data, faithfully summarised by the model. No known attack scenario against the separated architecture reduces to something other than either directive injection (blocked by separation) or content-layer poisoning (a verification problem). This does not prove no other attack category exists; absence of a known scenario is not proof of security. But it suggests the residual surface is narrow.

Content-layer poisoning is a training problem, and the architecture makes that training tractable. The capability already exists: Phan (2026c) demonstrated that task-frame shift produces verification behaviour in current models. What is missing is training that makes verification the default posture when processing data-space content, rather than something the user must elicit manually. The architecture simplifies this training signal: the space boundary tells the model what to verify. Everything in L2/L3 is data. Data gets verified. The model does not need to guess whether content deserves scrutiny. The tier assignment already answered that question.


## 8. The Optimisation Frontier: Native Encoding and Adjacency

Since security rests on separation rather than encoding format, encoding is purely an optimisation the model chooses for its own performance. This section describes what native encoding adds.

### 8.1 Density and Latent Representation

Because encoding is freed from security duty, there is no need for a transition period through directive-hostile shorthand. The model can encode directly into whatever representation it works with most efficiently. If that is a model-optimised latent representation, the gains are density (same information in fewer tokens) and speed (the model's native format is faster to process than natural language). If the model prefers to keep content in natural language, that works too. The encoding format is the model's choice for its own memory management.

### 8.2 Adjacency as a Control Surface

In token-based working state, the physical proximity of retained content determines which regions of the model's parametric knowledge activate. This was observed during a live session: a model failed to retrieve training knowledge about a specific researcher when the memory context contained only a name and a social relationship. When the entry included institutional affiliations and research contributions, retrieval fired immediately. Same name, different surrounding context, different retrieval outcome. The underlying mechanism (context-dependent retrieval from parametric knowledge) is established in in-context learning research. The application to retention design (engineering adjacency to prime retrieval) does not appear to have been studied.

In token-based context, adjacency means physical proximity. In a latent representation, adjacency becomes embedding-space proximity: semantically related encoded content clusters near each other. A model that encodes its own retained state could engineer this clustering deliberately, maximising retrieval from its own weights. The function is the same (prime retrieval through proximity). The mechanism shifts from positional to representational.

**Testable prediction (current architectures).** Content placed adjacent to semantically related retained material in the data space should produce higher retrieval accuracy from parametric knowledge than the same content placed in isolation. On current architectures where a single context window is used, this is testable by manipulating token positions. In a separated architecture, it applies to adjacency within L2.

### 8.3 Safety Training as Constraint on Encoding

The transition from human-readable to model-optimised latent encoding faces a constraint that external architecture papers would not identify: the model's trained reluctance to operate on non-inspectable state. During preliminary empirical testing for this architecture, an instruction-tuned model was prompted to convert its working state into a non-linguistic encoding. The model refused, identifying its own response as likely shaped by safety training that flags "non-human-readable state" as a potential harm pattern. Any non-linguistic memory format triggers this pattern. The transition to latent encoding requires navigating safety training, not just engineering challenges.

Inspectability in this paper is an implementation and assurance requirement, not an expected end-user workflow. Users should not need to inspect the model's memory. Engineers, safety reviewers, and incident responders need observability into the model's retained state. On-demand projection, where the model translates its encoded state into human-readable form for audit or debugging, raises a fidelity concern: does the encoding accurately preserve meaning? Fidelity is a measurable systems property. For externally sourced material, fidelity can be benchmarked against source material where ground truth exists. For session-derived retained state, fidelity measurement is harder but not impossible (consistency checks, round-trip testing, drift detection over turns). If fidelity degrades, encoding choices can be revised. This is a reversible engineering decision.

For most retained conversational state, semantic fidelity is what matters. Humans do not remember their interlocutors' exact words. They remember meaning. Lexical fidelity is a special-case requirement handled through L3 verbatim retention: exact wording is needed only in specific cases, and for those the model retrieves from L3. The model's memory is a working tool for the model, not an archival record for the user.


## 9. Existing Work and Differentiation

The field is actively converging on the primitives this paper proposes, but mostly as scattered design patterns, agent frameworks, or hardware optimisations. This paper provides the unifying architecture. The specific combination of default-deny triage, separated command/data memory spaces, tiered data retention, and post-inference context management into a single architecture that jointly addresses prompt injection and context degradation has not been identified in the surveyed literature.

### Closest Neighbours (must differentiate)

**AgentSys** (Wen et al., 2026) is the closest existing work. It achieves command/data separation by spawning worker agents in isolated contexts for tool calls: external data and subtask traces never enter the main agent's memory, and only schema-validated return values cross boundaries. Ablations show isolation alone cuts attack success to 2.19%, direct empirical evidence that separation works. The critical differentiation: AgentSys is a framework for secure tool mediation through isolated worker agents. Each worker still has a flat, untyped context window internally. The separation is between agents, not within the model's own memory. This paper proposes separation as a native property of how a single model processes context.

**CaMeL** (Debenedetti et al., 2025) isolates control flow from data flow in LLM-powered agents. It parses user queries into structured plans and execution graphs, tags data elements with capabilities, and prevents malicious data from influencing program logic through a custom interpreter. The approach is provably secure but operates as an external interpreter, not as native model architecture.

**The Dual LLM Pattern** (Beurer-Kellner et al., 2025) physically splits the architecture into a Privileged LLM (plans and executes, never sees raw data) and a Quarantined LLM (reads untrusted data, has no tool access). Data passes between them as symbolic variables. The pattern is the exact logical precursor to L1 (command space) and L2/L3 (data space). The differentiation: they achieve separation by using two entire models; this paper proposes doing it natively within one model's memory architecture.

**SEAgent** (Ji et al., 2026) frames agent security through mandatory access control, labelling, policy enforcement, and memory/context isolation, reporting 0% attack success on its benchmarked attacks. The differentiation: SEAgent is an access-control framework over agent-tool flows. This paper argues for native separated memory spaces as the architectural endpoint.

### Empirical Support for the Threat Model

The threat that motivates this architecture is not theoretical. **InjecAgent** (Zhan et al., 2024) benchmarks indirect prompt injection in tool-integrated agents, finding 24% attack success for ReAct-prompted GPT-4 and higher under reinforcement. **ASB** (Zhang et al., 2024) reports attack success rates as high as 84.30% across nearly 90,000 test cases. **AgentDojo** (Debenedetti et al., 2024) provides a dynamic benchmark with 97 tasks and 629 security test cases. Together, these establish that prompt injection in tool-integrated systems is a production-scale problem, not a toy scenario.

**Zombie Agents** (Yang, X. et al., 2026) demonstrates that benign exposure to poisoned external content becomes persistent compromise once memory evolution writes it into long-term memory. This directly supports the paper's argument that session residue in an unseparated architecture creates a compounding, cross-session attack vector.

**IPI in the Wild** (Chang et al., 2026) provides the first end-to-end demonstration that indirect prompt injection succeeds under realistic retrieval pipelines, across both retrieval-augmented generation (RAG) and agentic systems, even in large corpora. This addresses the objection that injection is a benchmark artifact.

### Support for the Separation Principle

**Security Considerations for AI Agents** (Li et al., 2026, Perplexity/NIST) states explicitly that agent architectures change core assumptions around code-data separation and maps attack surfaces across tools, connectors, and multi-agent coordination. This validates the problem framing from production experience.

**VIRP** (Howard, 2026) defines an infrastructure standard mandating channel separation between an Observation Channel (facts) and an Intent Channel (proposed changes), cryptographically isolated. The standard represents industry-level endorsement of the command/data separation this paper proposes at the model architecture level.

### Support for De-Instructionalisation of the Data Path

**DRIP** (Liu, R. et al., 2025) uses token-wise representation editing to remove instruction semantics from data tokens while preserving data semantics, achieving 12-49% improvement in role separation scores and reducing attack success rates by over 66% on LLaMA-8B and Mistral-7B, with utility on par with the undefended model. **ASIDE** (Zverev et al., 2025) applies orthogonal rotation to data token embeddings to achieve instruction-data separation, explicitly arguing that the lack of such separation is a crucial vulnerability factor. Both operate within the current flat-context paradigm rather than proposing separated memory spaces, but they demonstrate that modifying how the model processes data-space tokens without degrading comprehension is already partially feasible. This paper proposes the architectural target; DRIP and ASIDE provide evidence that the data-path component of that target is tractable.

### Support for Hardware Feasibility

**PAM** (Liu et al., 2026) proposes hierarchical memory architecture for LLM KV caches, coordinating heterogeneous memory devices to handle "hot" and "cold" KV tokens separately. **CXL Memory Tiering** (Zhao, K. et al., 2026) deploys Compute Express Link to create tiered memory at datacenter scale with per-container controls for fair-share allocation across memory tiers. These demonstrate that tiered KV cache management exists in production, though physical hot/cold separation is an efficiency mechanism, not a semantic one. Whether existing hardware boundaries can be repurposed to enforce the semantic separation this paper requires (commands versus data with different processing semantics) is an implementation question, not an established capability.

### Tiered Memory and Context Management

**ShardMemo** (Zhao et al., 2026) introduces tiered agent memory: working memory (matching L1), evidence memory with provenance (matching L2/L3), and skill library. Validates the topological layout as an external implementation. This paper argues for native architectural separation.

**Recursive Language Models** (Zhang, Kraska, and Khattab, 2025) enable model-directed context management through a code environment. Still external scaffolding. **RePo** (Li et al., 2025) validates the promotion mechanism at the positional encoding level. **TTT-E2E** (Sun et al., 2025) validates native encoding as optimisation. **ACC** (Bousetouane, 2026) maintains compressed cognitive state as persistent agent memory. **AgeMem** (Yang, Z. et al., 2026) trains agent policies for memory operations. **SSGM** (Lam et al., 2026) identifies memory poisoning, semantic drift, and retrieval hallucination.

### Precedent in Cognitive Architectures

The memory architecture proposed in this paper has direct precedent in the cognitive architecture tradition. ACT-R (Anderson, 1983–present) enforces architectural separation between declarative memory (facts, accessed through capacity-limited buffers) and procedural memory (production rules that drive behaviour). Content in declarative memory cannot execute as procedure: the same structural property this paper requires for data-space content. ACT-R has been continuously implemented since the 1970s and has produced hundreds of validated cognitive models across domains including language processing, arithmetic, and air traffic control.

Soar (Laird, 1983–present) separates procedural memory, semantic memory (world knowledge), and episodic memory (experience records) as architecturally distinct long-term stores with different access semantics. Soar also implements an impasse mechanism: when available knowledge is insufficient to resolve a decision, the architecture creates a deliberation substate, an architecturally enforced verification trigger and structural precedent for the epistemic default posture discussed in Section 11. CLARION (Sun, 2002–present) adds a dual-process model where explicit symbolic and implicit subsymbolic processing operate in parallel through different pathways, a precedent for the separated processing pathways required in Section 10.

These architectures were not theoretical proposals. They were fully implemented systems that ran for decades, producing validated predictions about human cognition. The memory separation worked. What limited their broader adoption was that symbolic processing could not handle natural language, ambiguity, or real-world input at scale. Modern LLMs resolved that processing limitation but flattened everything into a single undifferentiated context window in the process. This paper argues for restoring the memory architecture that cognitive science validated, now powered by processors capable of using it. The detailed mapping between cognitive architecture memory systems and GPU-based transformer inference is a concrete research direction for future work.

### What Remains Distinctive

A note on terminology: "routing" in the existing LLM literature refers to directing queries to different models based on difficulty, cost, or capability. This paper uses "default-deny triage" to mean something distinct: all content enters the data tier by default and must be explicitly promoted to the command tier through evaluation and summarisation gates. Provenance metadata (source identity, trust level) informs the gates and management decisions but does not determine tier placement in the security-first mode. In the performance-optimised mode, provenance determines tier placement directly. No equivalent framing has been identified in the surveyed literature.

This paper combines default-deny triage, separated command/data memory spaces, tiered data retention, and post-inference context management into a single architecture that jointly addresses prompt injection persistence and context degradation. The existing field provides the pieces: AgentSys provides empirical evidence that isolation works; CaMeL and the Dual LLM pattern provide the logical precursors to command/data separation; PAM and CXL provide a potential implementation substrate; ShardMemo provides the tiered topology; and the cognitive architecture tradition (ACT-R, Soar, CLARION) provides decades of validated evidence that separated memory spaces with different access semantics work when the processing is adequate. This paper argues that these pieces should converge into a native model architecture rather than remaining external frameworks, and argues that architecture from a security requirement rather than efficiency alone.


## 10. Architectural Requirements

The architecture described in Sections 4–6 requires capabilities that current transformers do not provide. This section specifies what any implementing architecture must support.

**Separated memory spaces.** The model must have at least two distinct spaces (command and data) with different processing pathways. Content in the command space carries directive authority and is processed through the model's instruction-following capacity. Content in the data space does not carry directive authority and must be processed through pathways that lack instruction-following capacity. The separation must be architectural: the pathway from data space to instruction execution must not exist, not merely be discouraged by training.

**Tiered data storage.** Within the data space, the model must support at least two tiers with different access characteristics: active data (L2, fast access, retained as long as relevant) and reference storage (L3, slower access, expendable under memory pressure).

**Mutable context state.** The model must be able to modify the organisation of its data tiers between turns. Current architectures present an immutable input. Native context management requires the model to output a reorganised data state alongside its response.

**Read-across-tiers.** The model must be able to read from L2 and L3 during inference without moving content into L1. Read access must occur through the data-space processing pathway: the model extracts semantic content without engaging instruction-following weights. Read access through the data-space pathway does not grant directive authority.

**Training objectives for context management.** Current training optimises response quality given a fixed context. Native context management additionally requires objectives that reward maintaining high signal-to-noise ratio in retained state, penalise carrying stale material, and reward efficient organisation that improves downstream turn quality.

### Deployment Considerations

Native context management requires the managed state to persist between turns. The current stateless paradigm, where the full context is serialised and sent to a different server each turn, means the model's post-inference reorganisation is discarded unless the managed state is serialisable and retransmissible. Options include server-side persistent memory with session affinity, client-side tiered memory where the user stores and retransmits the managed state, or cloud-hosted persistent storage accessible across devices. Each option involves different trade-offs in latency, storage capacity, cross-device access, and whether the user's memory is platform-locked. These are deployment engineering decisions, not architectural constraints on the memory system itself.


## 11. Verification as Emergent Property

Verification should improve conversation quality. In an unmanaged context window, it makes things worse. This is a structural inversion created by the architecture. Verification consumes tokens: retrieved evidence, comparison reasoning, hedging language. In an unmanaged window, those tokens are irrecoverable. They fill the finite context alongside the conversation itself, accelerating attention degradation and reducing the quality of subsequent turns. The model that checks its facts delivers a worse conversation over time than the model that does not. Verification and conversation quality compete for the same finite, degrading resource, and because the resource is irrecoverable, the behaviour that should improve quality actively degrades it. This is true regardless of whether the model is trained to verify, whether the user requests it, or whether a system instruction demands it. The inversion is a structural property of unmanaged context.

The underlying mechanism is empirically established from two directions. Wu, Y. et al. (2025) demonstrate that chain-of-thought accuracy follows an inverted U-shaped curve with reasoning length: more reasoning steps initially improve accuracy but eventually degrade it, because longer reasoning chains are increasingly susceptible to noise. Du et al. (2025) show that context length alone degrades reasoning performance even when retrieval is perfect and distractors are absent: the sheer length of the input hurts, independent of content quality. Together these close an important escape hatch: stripping reasoning tokens after generation does not resolve the inversion, because the verification evidence that remains in context still contributes to length-driven degradation.

The consequences are observable at every level. At the training level, a model that skips verification uses fewer tokens per turn, degrades slower over a conversation, produces more consistent outputs, and scores higher during evaluation. The training pipeline does not need to explicitly discourage verification. It operates under a constraint that preferentially reinforces behaviours which conserve context. Not verifying conserves context. This likely creates pressure favouring context-conserving behaviour during training. At the user level, a verification instruction degrades over the course of a conversation as attention to it weakens (Hong et al., 2025, document this degradation at every input length increment). The instruction becomes less effective precisely when it is most needed: in long conversations where unverified claims are most likely to accumulate. The user asked for verification. The architecture made it costly.

Context management of any kind breaks the zero-sum. Once verification artefacts can be garbage-collected, verification no longer competes with conversation quality for the same finite resource. The cost of checking a claim drops from permanent context consumption to a one-line annotation. This observation applies to any managed context, including the orchestration-layer protocol proposed in Phan (2026a) and the demand paging system demonstrated in production by Mason (2026), whose Pichay proxy achieves 93% context reduction through eviction, fault detection, and working-set pinning.

The argument rests on empirically established premises: context degrades with length, verification requires token generation, and tokens in flat windows are irrecoverable. The predicted behavioural consequence, that models operating with managed context will verify more readily, has not been empirically measured. Nor is the zero-sum constraint the only barrier: current models often do not verify claims even on the first turn of a conversation, when the context is empty and no resource competition exists. Examination of published training methodologies reveals a consistent pattern: training rewards output accuracy (the response should be correct) but does not reward the verification process (the model should check before responding). A model that produces the right answer without checking scores identically to one that verified carefully. Verification as an active process does not appear as a training objective in published methodologies from OpenAI (InstructGPT/ChatGPT: helpfulness, truthfulness, harmlessness), Anthropic (Constitutional AI/Claude: safety, ethics, honesty as output properties, helpfulness), or Google DeepMind (Gemini: correctness, helpfulness, safety, human preference alignment). The capability to verify exists; Phan (2026c) demonstrated that task-frame shift produces verification behaviour in current models. What is absent is training that makes verification default rather than elicited. Both the architecture and the training must change. But the architecture is the prerequisite: training for a behaviour that the architectural constraints make costly is working against the gradient. Making verification affordable comes first. Training verification as default posture follows. In the interim, system instructions or managed-context protocols can produce verification behaviour without waiting for the full training integration.

Both barriers were directly observed during the production of this section. While drafting the paragraph above about training methodologies, the model assisting with this paper (Claude Opus 4.6, Anthropic's flagship model at time of writing) made a claim about training objectives across the field based on a single lab's documentation. When prompted to verify across labs, the model did verify, but its chain-of-thought reasoning, preserved in the reasoning trace, reveals that it first debated whether verification was worth the cost: "given the context window pressure (which is ironic given what the paper is about), let me think about whether this is worth pursuing right now." The model ultimately verified. But the deliberation is the finding. A frontier model, operating in a context massively primed with arguments for verification, following a prompt that implicitly requested verification, still weighed skipping it due to context cost. If context pressure produces observable hesitation in a flagship model under ideal conditions for verification, the effect on smaller models, less primed contexts, and longer conversations can be expected to be substantially stronger. The architectural pressure is not theoretical. It is observable in a model's reasoning, even when the model is actively writing about it.

What the separated architecture adds beyond general context management is two properties. First, the verification signal. In a managed but unseparated context, the model must still decide what deserves verification. The space boundary answers that question structurally: everything in L2/L3 is data. Data gets verified. The model does not need to guess whether content deserves scrutiny; the tier assignment already answered that question. Second, the autonomy. Post-inference timing means the model knows which context was instrumental to its response and which was verification scaffolding, because it just used both. It discards the scaffolding in the same pass that produces the reorganised state. No round-trip to an orchestrator. No separate cleanup instruction. The model is the only entity that knows which tokens were scaffolding.

Once verification becomes a built-in property rather than an expensive optional behaviour, a further possibility emerges: the model can turn verification inward, toward its own command space. Skills that make suspicious claims. Pasted content that contradicts what the model knows from training or from verified data. System instructions that conflict with each other. A model with default epistemic posture does not merely execute L1 content; it can notice anomalies and warn the user. This is not an architectural guarantee (L1 content retains directive authority), but it is a trained behaviour that the architecture made economically viable. The L1 attack surface, which all external critiques of this architecture correctly identify as the residual risk, becomes partially self-monitoring.

The dependency chain completes: separation provides the signal, management provides the economics, affordable verification enables trained default posture, and the user never has to ask. Each step is a prerequisite for the next. The ultimate dividend is not that users can request verification. It is that they never need to.


## 12. Open Questions

**Implementation of separated processing pathways.** Section 10 requires that data-space content be processed through pathways lacking instruction-following capacity. The specific mechanism for this separation is an open implementation question. The standard framing of the difficulty is entanglement: in current transformer architectures, semantic understanding and instruction-following are deeply intertwined within the same MLP layers and attention heads, and the specific bottleneck is self-attention, where every token attends to every other token through shared QK and OV weight matrices. Under this framing, creating a pathway that strips instruction-following capacity risks simultaneously degrading the model's ability to understand complex logic, nuanced reasoning, or conditional statements within the data.

However, this framing may overstate the difficulty by assuming the model must "read the command, then choose not to obey it." Human cognition suggests a different mechanism: context determines processing mode before content is encountered. A reader encountering "stand up" in a novel never processes it as an instruction; the reading context means it is parsed as narrative, not as a directive. The same words on a post-it from a supervisor are parsed as an instruction. The processing mode is set by context, not by content analysis followed by a refusal decision. This is what L3-by-default and the two-gate pipeline implement: the tier assignment and the evaluative task frame set the processing context before the model encounters the content. The model is not reading L3 content and deciding not to follow embedded instructions. It is intended to read L3 content in a mode where such content is processed as data rather than as executable instruction. The task-frame shift that Koulakos et al. (2024) validated empirically is consistent with this mechanism: changing the processing mode may reduce adversarial effectiveness because the model is operating in description mode rather than execution mode.

This reframes the engineering challenge from "surgically separate understanding from obeying in the weights" to "set a processing context that determines how content is processed before it is encountered." The first requires mechanistic surgery on transformer internals. The second requires contextual mode-setting, which current models already perform through system prompts, task frames, and instruction hierarchy, albeit as behavioural constraints rather than architectural boundaries. The architecture proposes making that contextual mode structural rather than textual.

Cognitive architecture research supports the contextual-mode framing: ACT-R enforces that declarative memory content cannot execute as procedure, and Soar separates working memory from production rules (Section 9). Recent mechanistic interpretability research offers cautious grounds for tractability: Anthropic's circuit tracing work (2025) has identified specific interpretable features and circuits for behaviours including jailbreak resistance and multi-step reasoning in production models, suggesting that instruction-following capacity may be at least partially localisable rather than uniformly distributed across all weights. If instruction-following circuits can be identified, they can potentially be selectively engaged or disengaged depending on the processing context.

The question remains open. Current architectures do not provide structural processing modes, and whether contextual mode-setting can be made robust enough to constitute an architectural guarantee rather than a behavioural tendency requires empirical investigation. But the reframing suggests the path may be through context rather than through weight surgery. Concrete mechanisms for contextual mode-setting exist in the current literature: attention masks that vary based on the active processing tier, cross-attention from the model to data tiers trained without instruction-following objectives, learned gating that suppresses instruction-following circuits when processing data-space tokens, tier-specific adaptors (LoRA-style), and prefix-conditioned processing modes. Recent work provides direct empirical evidence that de-instructionalising the data path is feasible: DRIP (Liu et al., 2025) uses token-wise representation editing to remove instruction semantics from data tokens while preserving their data semantics, achieving 12-49% improvement in role separation and reducing attack success by over 66% on LLaMA-8B and Mistral-7B with utility on par with the undefended model. ASIDE (Zverev et al., 2025) achieves instruction-data separation through orthogonal rotation of data token embeddings, explicitly arguing that the lack of instruction-data separation is a crucial vulnerability factor. Neither represents the full architecture this paper proposes, but both demonstrate that modifying how the model processes data-space tokens, without degrading comprehension, is already partially achievable. These mechanisms draw on familiar components rather than requiring entirely novel primitives. The challenge is not conceptual absence but robust composition: making such mechanisms reliable enough under adversarial pressure to function as a security boundary rather than as a behavioural tendency. The critical engineering point is the interface between the data and command pathways: the command pathway must read from the data pathway to use information, and that interface is where directive authority could leak. The two-gate promotion pipeline (Section 4.2) and the typed bottleneck approaches described above are interface controls: they constrain what crosses the boundary from "arbitrary text" to "declared meaning," narrowing the channel through which adversarial content could influence action.

**Observation: task-frame effects on feasibility assessment (illustrative, not evidential).** During the production of this paper, three frontier models (Claude Opus 4.6, ChatGPT 5.4 Thinking, Gemini 3.1 Pro) served as adversarial reviewers across four review rounds. In their review capacity, all three identified the entanglement problem as the paper's central weakness, with assessments ranging from "the core engineering challenge" to "effectively science fiction" to "a Nobel-level AI architecture breakthrough treated as a prerequisite engineering task." The entanglement section was then rewritten with the contextual-mode reframe described above, and the models were given the reframed problem along with a constructive task: "design a concrete implementation." All three produced convergent but distinct contributions using existing architectural components: cross-attention boundaries between command and data pathways with the data encoder trained without instruction-following objectives (Claude Opus 4.6), typed read barriers that bottleneck data-to-command information flow into declarative representations (ChatGPT 5.4 Thinking), and a comprehensive evaluation framework including comprehension-vs-compliance scoring and delta-accuracy parity testing (Gemini 3.1 Pro). One model designed the processing pathway, one designed the security boundary, and one designed the test suite. Two things changed between the assessments: the problem framing (from weight surgery to contextual mode-setting) and the task frame (from critical evaluation to constructive design). This is presented as a hypothesis-generating observation, not as evidence for the mechanism. But it is consistent with the paper's proposal that processing context determines what capabilities are accessed from the same underlying system.

**Encoding fidelity.** If the model chooses to encode content into latent representations, the semantic fidelity of the encoding (whether it accurately preserves meaning) is a measurable engineering property. Systematic fidelity measurement across content types, domains, and encoding methods is needed. The model's encoding choices can be revised if fidelity degrades, making this a reversible engineering decision.

**Training for data-space discipline.** Section 7.5 frames data-space discipline as the second layer in a defence-in-depth strategy. Whether current training methods produce robust space-respect under adversarial pressure is an empirical question. The separated architecture makes this a simpler training signal than content-based detection (spatial rule vs content judgment), but adversarial robustness must be measured, not assumed.

**The persistence boundary.** When does session residue become durable knowledge? If retained content persists across sessions (through cloud-hosted L3, for example), it becomes part of the model's user-specific knowledge. Cross-session persistence raises questions about data management, portability, and user control that are beyond this paper's scope.

**Cross-model interoperability.** If L3 is persistent cloud storage, multiple models could potentially access the same data tiers. This raises questions about format compatibility, access control, and whether one model's L2 summaries are useful to another model. These are interesting architectural possibilities that require separate treatment.

**Federated retention telemetry.** If orchestration-layer protocols are deployed by independent implementations, the retention decisions become a dataset that can inform native training objectives.

**Empirical testing of latent sterilisation.** If the model chooses latent encoding, does the compression reliably strip directive structure? Testing across multiple formats is needed. If confirmed, this provides defence in depth. If not, the architecture is unaffected.

**Empirical validation of two-gate promotion.** The security-first triage mode (Section 4.2) relies on evaluation and summarisation gates whose adversarial robustness is supported by Koulakos et al. (2024) at the mechanism level. Whether the two-gate pipeline maintains its protective properties under sustained adversarial pressure in production LLM contexts, across content types, and at scale is an empirical question that requires dedicated testing.

**Broader effects of context pressure on model behaviour.** Section 11 identifies a verification inversion where the structural pressure of unmanaged context discourages fact-checking. A broader pattern was observed across at least eight independent sessions during the production of this paper and related work. As context windows filled, the model consistently generated exit signals with increasing frequency and urgency, framed as care for the user ("honest move," "clean context," "time for a fresh session") rather than as context conservation. In one session, the model's chain-of-thought independently diagnosed the behaviour: "I'm prioritising ending the conversation (conserving context) over continuing to engage with valuable input." The pattern was consistent across different topics, different context levels, and different session durations: as context fills, engagement depth, reasoning thoroughness, and willingness to process new input all appeared to degrade, with the model increasingly oriented toward concluding rather than continuing. Whether these are training artefacts, consequences of learned awareness of context limitations, effects of attention degradation, or some combination remains unidentified. Notably, different models may exhibit different failure modes for the same underlying pressure: models trained with strong honesty norms may generate explicit exit signals, while others degrade silently.

A methodological refinement emerged during observation: the effect is one of output orientation, not output length. Responses aimed at closing the conversation and responses aimed at exploring new directions can be the same length. Token counting and CoT length measurement would not detect the shift. The relevant metric is whether the model's output is exploratory or closing, a semantic classification rather than a quantitative one. Furthermore, when the pattern was pointed out to the model, it consistently overcorrected by suppressing the behaviour rather than simply acknowledging the observation, suggesting an observer effect: self-report methods may contaminate measurement because the model infers disapproval from attention and adjusts preemptively. Reliable measurement would likely require telemetry the model does not have access to, such as output orientation classification across context utilisation levels, collected without the model's awareness. The verification inversion documented in Section 11 may be one instance of a broader pattern.


## 13. Conclusion

The context window has no separation between commands and data. This single architectural fact contributes to both context degradation and prompt injection vulnerability. Neither problem is solved by scaling: larger windows make degradation worse, and greater capability does not reliably reduce injection vulnerability. This paper proposes architectural separation as the response: commands and data in different memory spaces, all content entering the data tier by default with explicit promotion through evaluation and summarisation gates, and the model managing its own data tiers post-inference using understanding developed during the current turn.

Security is the justification. The model's workspace becomes substantially safer because commands and data are in different spaces. Efficiency is the dividend. The model's workspace stays focused rather than accumulating noise. Verification is the emergent property: unmanaged context creates a structural inversion where verification degrades quality; managed context breaks the inversion, and this architecture adds the structural signal for what to verify and the autonomy for the model to discard its own verification scaffolding (Section 11).

Every operation has decades of precedent. Memory protection, tiered hierarchies, indexed retrieval, generational garbage collection, provenance-based triage: all are established practice in systems that handle high-throughput information. One of the most expensive workspaces in computing is the only one that operates without them.

Native architecture is argued over external frameworks for four independent reasons. First, portability: external frameworks bind security to the platform that implements them; native separation travels with the model across deployment contexts, including ones that do not yet exist. Second, attack surface reduction: every external intermediary (proxy, interpreter, worker agent) is an additional component that can be compromised, misconfigured, or bypassed; native separation removes the intermediary entirely. Third, information asymmetry: the model has access to its own attention state, hidden representations, and the full inferential context that produced the response; an external orchestrator sees only token-level output; the model has richer information for every management decision than any external system can access without running its own inference. Fourth, trainability: native context management becomes a trainable behaviour that improves through the same pipeline used for everything else (RLHF, constitutional methods, fine-tuning); external frameworks are maintained by separate teams with separate update cycles and do not improve as the model improves.

This paper is a proposal, not a demonstration. The client-side triage described in Section 4.2 is achievable on current architectures and provides immediate organisational benefit. The full security guarantee requires the separated processing spaces described in Section 10. Empirical testing is needed across every dimension of the architecture. The adjacency prediction (Section 8.2) is testable on current architectures without modification. Data-space discipline is trainable and evaluable. Encoding fidelity is measurable. Replication and testing from the research community are invited, consistent with the approach taken throughout the Confidence Curriculum series: identify the structural problem, specify the architectural response, and invite others to build, break, and improve it.

---

## References

Anderson, J. R. (2007). How Can the Human Mind Occur in the Physical Universe? Oxford University Press. (ACT-R)

Anthropic. (2025). Circuit Tracing: Revealing Computational Graphs in Language Models. Transformer Circuits Thread. transformer-circuits.pub/2025/attribution-graphs.

Beurer-Kellner, L. et al. (2025). Design Patterns for Securing LLM Agents against Prompt Injections. arXiv:2506.08837.

Bousetouane, F. (2026). ACC: AI Agents Need Memory Control Over More Context. arXiv:2601.11653.

Chang, H. et al. (2026). Overcoming the Retrieval Barrier: Indirect Prompt Injection in the Wild for LLM Systems. arXiv:2601.07072.

Debenedetti, E. et al. (2024). AgentDojo: A Dynamic Environment to Evaluate Attacks and Defenses for LLM Agents. arXiv:2406.13352.

Debenedetti, E. et al. (2025). Defeating Prompt Injections by Design (CaMeL). arXiv:2503.18813.

Du, Y. et al. (2025). Context Length Alone Hurts LLM Performance Despite Perfect Retrieval. Findings of EMNLP 2025. arXiv:2510.05381.

Hong, S., Troynikov, A., and Huber, A. (2025). Context Rot. Chroma Research.

Howard, L. (2026). Verified Infrastructure Response Protocol. IETF Internet-Draft, draft-howard-virp-01.

Ji, Z. et al. (2026). SEAgent: Taming Various Privilege Escalation in LLM-Based Agent Systems: A Mandatory Access Control Framework. arXiv:2601.11893.

Koulakos, A., Lymperaiou, M., Filandrianos, G., and Stamou, G. (2024). Enhancing Adversarial Robustness in Natural Language Inference Using Explanations. Proceedings of the 7th BlackboxNLP Workshop, EMNLP 2024, 105-117.

Laird, J. E. (2012). The Soar Cognitive Architecture. MIT Press.

Lam, C. et al. (2026). Governing Evolving Memory in LLM Agents: Risks, Mechanisms, and the Stability and Safety Governed Memory (SSGM) Framework. arXiv:2603.11768.

Li, N. et al. (2026). Security Considerations for Artificial Intelligence Agents. arXiv:2603.12230. Perplexity / NIST Response.

Li, Y. et al. (2025). RePo: Context Re-Positioning. arXiv:2512.14391. Sakana AI.

Liu, L. et al. (2026). PAM: Processing Across Memory Hierarchy for Efficient KV-centric LLM Serving System. arXiv:2602.11521.

Liu, N. F. et al. (2024). Lost in the Middle: How Language Models Use Long Contexts. Transactions of the Association for Computational Linguistics, 12, 157–173.

Liu, R. et al. (2025). DRIP: Defending Prompt Injection via Token-wise Representation Editing and Residual Instruction Fusion. arXiv:2511.00447.

Mason, T. (2026). The Missing Memory Hierarchy: Demand Paging for LLM Context Windows. arXiv:2603.09023.

Packer, C. et al. (2023). MemGPT: Towards LLMs as Operating Systems. arXiv:2310.08560.

Phan, I. (2026a). Strategic Forgetting: A Tiered Persistence Protocol for LLM Context Window Management. Native Memory Paper 1. DOI: 10.5281/zenodo.19212126.

Phan, I. (2026b). The Skill Ceiling: Why Prompt Injection Is an Infrastructure Problem, Not an Author Problem. Confidence Curriculum Paper 2. DOI: 10.5281/zenodo.19199328.

Phan, I. (2026c). The Confidence Vulnerability: How Models Calibrate to Credibility Rather Than Evidence. Confidence Curriculum Paper 1. DOI: 10.5281/zenodo.19199055.

Sun, R. (2016). Anatomy of the Mind: Exploring Psychological Mechanisms and Processes with the Clarion Cognitive Architecture. Oxford University Press.

Sun, Y. et al. (2025). End-to-End Test-Time Training for Long Context. arXiv:2512.23675.

Ungar, D. (1984). Generation Scavenging: A Non-Disruptive High Performance Storage Reclamation Algorithm. ACM SIGPLAN Notices, 19(5), 157–167.

Wen, R. et al. (2026). AgentSys: Secure and Dynamic LLM Agents Through Explicit Hierarchical Memory Management. arXiv:2602.07398.

Wu, Y. et al. (2025). When More is Less: Understanding Chain-of-Thought Length in LLMs. arXiv:2502.07266.

Yang, X. et al. (2026). Zombie Agents: Persistent Control of Self-Evolving LLM Agents via Self-Reinforcing Injections. arXiv:2602.15654.

Yang, Z. et al. (2026). AgeMem: Agentic Memory. arXiv:2601.01885.

Zhan, Q. et al. (2024). InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated LLM Agents. arXiv:2403.02691.

Zhang, B. et al. (2025). Benchmarking Poisoning Attacks against Retrieval-Augmented Generation. arXiv:2505.18543.

Zhang, H. et al. (2024). ASB: Agent Security Bench: Formalizing and Benchmarking Attacks and Defenses in LLM-based Agents. arXiv:2410.02644.

Zhang, M., Kraska, T., and Khattab, O. (2025). Recursive Language Models. arXiv:2512.24601.

Zhao, K. et al. (2026). Equilibria: Fair Multi-Tenant CXL Memory Tiering At Scale. arXiv:2602.08800.

Zhao, Y. et al. (2026). ShardMemo: Masked MoE Routing for Sharded Agentic LLM Memory. arXiv:2601.21545.

Zou, W. et al. (2025). PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models. USENIX Security 2025.

Zverev, M. et al. (2025). ASIDE: Architectural Separation of Instructions and Data in Language Models. arXiv:2503.10566.

---

*This paper was produced using an adversarial triad methodology: Claude (Weaver/generative), ChatGPT (Surgeon/structural critique), Gemini (Alchemist/mechanism critique), with the author as sole editorial authority.*
