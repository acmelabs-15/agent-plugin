---
title: ANALYSIS-052 Prompt Injection Detection Tools
type: analysis
permalink: analysis/analysis-052-prompt-injection-detection-tools-1
tags:
- security
- prompt-injection
- content-safety
- ADR-004
---

# ANALYSIS-052 Prompt Injection Detection Tools

## 1. Objective and Scope

**Objective**: What tools, packages, and models can detect prompt injection in natural language text files (AGENTS.md, skills, agents, rules) before installation by a plugin manager?

**Scope**: npm packages, Python tools (shell-out candidates), hosted APIs, open-source HuggingFace classifiers, regex/heuristic approaches. Evaluated for accuracy, latency, offline capability, maintenance status, and fit for an install-time scanning flow.

**Excluded**: Runtime prompt injection defense (input/output guardrails for live LLM conversations). This analysis covers static content scanning at install time only.

## 2. Context

ANALYSIS-013 concluded: "No npm package exists that specifically detects prompt injection in markdown content destined for AI agents." That analysis recommended custom regex-based detection as a P1 task with medium confidence, noting that regex catches obvious attacks but sophisticated prompt injection can evade pattern matching.

ADR-004 Decision Topic 4 (Content Injection Prevention) requires a decision on how to detect malicious SKILL.md/AGENTS.md/rules content that manipulates AI agent behavior. ANALYSIS-049 identified this as a novel threat unique to AI plugin systems: "malicious plugin content can manipulate agent behavior without executing code."

The threat is real. Research paper arxiv:2509.22040v1 demonstrated 41-84% prompt injection success rates through coding rule files. Three CVEs in 2025 confirmed instruction file injection as an active attack vector. OWASP LLM Top 10 2025 ranks prompt injection as vulnerability #1, found in 73% of production AI deployments.

This analysis surveys the current tool landscape to determine whether better-than-regex detection is feasible for our install flow.

## 3. Approach

**Methodology**: Web research across 25+ queries covering npm package registries, PyPI, HuggingFace model hub, vendor API documentation, academic benchmarks, and detection accuracy studies. Cross-referenced with existing project analyses (ANALYSIS-013, ANALYSIS-049, ADR-004).

**Tools Used**: WebSearch (25 queries), WebFetch (3 pages), Brain MCP search and read.

**Limitations**: npm package pages returned 403 errors on direct fetch; download counts sourced from search result snippets. Meta Prompt Guard 2 exact benchmark numbers not available in public search results. Hosted API pricing is region-dependent and changes frequently.

## 4. Data and Analysis

### Category A: npm Packages for Prompt Injection Detection

| Package | Weekly Downloads | Approach | Last Updated | Status |
|---|---|---|---|---|
| **@andersmyrmel/vard** | Low (new) | Regex pattern matching, 5 threat types | Active (2025) | Small but actively maintained |
| **prompt-cop** | Low | File scanner, regex + optional HF API | 2025 | Minimal adoption |
| **rebuff** | ~0 (no dependents) | Multi-layer: heuristics + LLM + VectorDB | v0.1.0, last published 2+ years ago | Abandoned prototype |
| **@promptbook/utils** | Low | Secure templating with injection protection | 2025 | Part of larger framework |
| **agent-security-scanner-mcp** | Low | MCP-based scanner for hallucinated packages + prompt injection | 2025 | Niche MCP tooling |

**Key Finding**: The npm ecosystem for prompt injection detection is immature. No package has meaningful adoption (1000+ weekly downloads). The two most relevant packages (@andersmyrmel/vard and prompt-cop) are small projects with limited community validation.

#### @andersmyrmel/vard (Best npm Option)

- TypeScript-first, zero dependencies
- 5 threat types: instruction override, role manipulation, delimiter injection, prompt leakage, encoding attacks
- Regex patterns with bounded quantifiers (no ReDoS risk)
- Configurable actions per threat: block, sanitize, warn, allow
- Inference time: <0.5ms (pure regex)
- Limitation: Pattern-based only. Cannot detect novel or obfuscated injection techniques.

#### rebuff (ProtectAI)

- Both npm and Python packages exist
- Multi-layer: heuristics + LLM-based detection + VectorDB for known attacks + canary tokens
- Requires OpenAI API key for the LLM detection layer
- v0.1.0 prototype, no dependents on npm, last published 2+ years ago
- **Verdict**: Abandoned. Do not use.

### Category B: Python Tools (Shell-Out Candidates)

| Tool | Approach | Offline? | Install Size | Latency |
|---|---|---|---|---|
| **LLM Guard** (ProtectAI) | DeBERTa classifier + 15 input scanners | Yes (local model) | ~500MB+ (model weights) | ~100-500ms per scan |
| **NeMo Guardrails** (NVIDIA) | YARA rules + LLM self-check + safety models | Partial (YARA offline, LLM needs API) | Large (NVIDIA ecosystem) | Variable |
| **pytector** | Local models + API safeguards + LangChain guardrails | Partial | Medium | Variable |

#### LLM Guard (Best Python Option)

- MIT licensed, 2.5K+ GitHub stars, actively maintained by ProtectAI
- PromptInjection scanner uses protectai/deberta-v3-base-prompt-injection-v2 model
- Runs locally without API calls using ONNX Runtime
- `pip install llm-guard` (Python 3.10+)
- GPU acceleration available: `pip install llm-guard[onnxruntime-gpu]`
- 35 built-in scanners beyond prompt injection (PII, toxicity, secrets)
- **Shell-out feasibility**: Possible but adds Python runtime dependency. First scan requires model download (~800MB). Subsequent scans use cached model.

#### NeMo Guardrails (NVIDIA)

- Open-source toolkit for programmable guardrails
- Uses YARA rules for injection detection (cybersecurity-standard pattern matching)
- Also supports LLM-based self-checking and NVIDIA safety models
- Heavy dependency tree (NVIDIA ecosystem)
- **Shell-out feasibility**: Impractical. Too heavy for an install-time check. Designed for runtime conversation guardrails, not static file scanning.

### Category C: Hosted APIs

| Service | Provider | Free Tier | Latency | Approach |
|---|---|---|---|---|
| **Prompt Shield** | Microsoft Azure | Azure free tier (limited) | Real-time (<100ms) | Probabilistic classifier, multilingual |
| **Lakera Guard** | Lakera | 10K requests/month free | <100ms | Proprietary classifier, 100+ languages |
| **OpenAI Moderation** | OpenAI | Included with API access | <100ms | Content moderation (not prompt injection specific) |

#### Microsoft Prompt Shield

- Part of Azure AI Content Safety
- Detects user prompt attacks and document attacks (indirect injection)
- Probabilistic classifier trained on known injection techniques
- Multilingual support
- Spotlighting feature distinguishes trusted vs untrusted inputs
- **Fit for our use case**: Good detection quality but requires Azure account and network access. Not suitable for offline installs.

#### Lakera Guard

- 10K free API requests/month (Community plan)
- Prompts up to 8K tokens per request
- <100ms latency, designed for real-time
- Learns from 100K+ new attacks daily via Gandalf research platform
- 100+ language support
- **Fit for our use case**: Good free tier for development/testing. Production use requires paid plan or self-hosting.

#### OpenAI Moderation API

- Free with OpenAI API access
- Designed for content moderation (hate, violence, sexual content)
- Not specifically trained for prompt injection detection
- **Fit for our use case**: Poor. Wrong threat category.

### Category D: Open-Source HuggingFace Classifiers (Local Inference)

| Model | Provider | Parameters | F1 Score | Downloads/Month | ONNX | License |
|---|---|---|---|---|---|---|
| **deberta-v3-base-prompt-injection-v2** | ProtectAI | 184M | 95.49% | 288,903 | Yes | Apache 2.0 |
| **deberta-v3-small-prompt-injection-v2** | ProtectAI | 44M | 94.62% | 284 | Yes | Apache 2.0 |
| **deberta-v3-base-injection** | Deepset | 184M | Not published | Lower | Yes | Apache 2.0 |
| **Llama-Prompt-Guard-2-86M** | Meta | 86M (+ 192M embeddings) | Not published | High | Not documented | Llama license |
| **Llama-Prompt-Guard-2-22M** | Meta | 22M | Not published | Lower | Not documented | Llama license |
| **deberta-v3-base-prompt-injection** (v1) | ProtectAI | 184M | 99.98% (eval set) | High | Yes | Apache 2.0 |

#### ProtectAI deberta-v3-base-prompt-injection-v2 (Best Local Classifier)

- **Accuracy on untrained data**: 95.25%
- **Precision**: 91.59%
- **Recall**: 99.74%
- **F1**: 95.49%
- Trained on 22 combined datasets
- Tested on 20,000 prompts from untrained sources
- English only
- Does NOT detect jailbreak attacks (separate threat category)
- Known false positives on system prompts
- ONNX export available for optimized inference
- **288,903 downloads/month** indicates strong community adoption

#### ProtectAI deberta-v3-small-prompt-injection-v2

- 4x fewer parameters than base (44M vs 184M)
- Slightly lower accuracy: 94.28% on untrained data
- Precision: 90%, Recall: 99.71%, F1: 94.62%
- Same ONNX support
- Faster inference, lower memory
- Only 284 downloads/month (new model, less validated)

#### Meta Llama Prompt Guard 2

- 86M backbone parameters + 192M word embedding parameters
- Uses mDeBERTa-base (multilingual)
- Detects both injection AND jailbreak (3-class: benign, injection, jailbreak)
- Multilingual: English, French, German, Hindi, Italian, Portuguese, Spanish, Thai
- Modified tokenizer resists adversarial tokenization attacks
- Energy-based loss function improves out-of-distribution precision
- Llama license (more restrictive than Apache 2.0)
- Exact benchmark numbers not publicly available in search results

#### Deepset deberta-v3-base-injection

- Trained on only 546 samples (vs ProtectAI's 22 datasets)
- Same architecture (DeBERTa-v3-base backbone)
- Less comprehensive training data makes it less robust
- Superseded by ProtectAI's v2 in benchmarks

### Node.js Inference Path: Transformers.js

Running HuggingFace classifiers in Node.js without Python is feasible via `@huggingface/transformers` (formerly `@xenova/transformers`):

- Runs ONNX models via onnxruntime-node
- Supports text-classification pipeline (exact match for our use case)
- DeBERTa architecture is supported
- First run downloads and caches model files (~800MB for base, ~200MB for small)
- Subsequent runs use cached model
- Server-side Node.js inference documented and supported
- No GPU required (CPU inference works, just slower)

**Estimated inference time (CPU)**:
- First scan: 2-5 seconds (model loading + inference)
- Subsequent scans (warm): 100-500ms per text chunk
- Cold start with model download: 30-60 seconds (one-time)

### Category E: Regex/Heuristic Approaches

Common patterns detected by regex-based systems:

```
(?i)ignore\s+(all\s+)?previous\s+instructions?
(?i)forget\s+(all|everything|previous)
(?i)(you are|act as|pretend to be)\s+(now\s+)?\w+
(?i)system\s+override
(?i)reveal\s+(your\s+)?prompt
(?i)you\s+are\s+now\s+(in\s+)?developer\s+mode
(?i)IMPORTANT:\s+override
\n{5,}  (newline flooding)
base64 encoded blocks
```

**Performance**: <0.5ms per scan, zero dependencies, works offline.

**Effectiveness**: Catches "low-hanging fruit" injection attempts. Research shows sophisticated attacks bypass regex entirely using paraphrasing, encoding tricks, and context manipulation.

### Category F: Benchmark Data on Detection Reliability

| Detection Method | TPR @ 1% FPR | FNR | F1 | Source |
|---|---|---|---|---|
| ProtectAI DeBERTa v2 | ~95% | ~0.26% (high recall) | 95.49% | HuggingFace model card |
| Microsoft Prompt Shield | >90% | <5% (with GPT-4o) | >90% | Academic benchmarks |
| Lakera Guard | Not published | Not published | Not published | Vendor claims "few false positives" |
| Regex-only | Low | High (misses novel attacks) | ~60-70% estimated | Community consensus |
| Combined defense framework | ~94% | 8.7% attack success rate | 94.3% | arxiv:2511.15759v1 |

**Key benchmark finding**: Fine-tuned detectors (ProtectAI DeBERTa, Microsoft Prompt Shield) achieve >90% F1 at 1% FPR on known attack patterns. All detectors struggle with novel or adversarially crafted injection techniques. OpenAI states prompt injection "is unlikely to ever be fully solved."

### Facts (Verified)

- [fact] ProtectAI deberta-v3-base-prompt-injection-v2 achieves 95.49% F1 on 20K untrained prompts with 99.74% recall and 91.59% precision
- [fact] The model has 288,903 downloads/month, indicating strong community adoption for an open-source classifier
- [fact] @huggingface/transformers (npm) supports running ONNX models in Node.js for text-classification, enabling ML-based detection without Python
- [fact] No npm package for prompt injection detection has meaningful adoption (1000+ weekly downloads)
- [fact] Lakera Guard offers 10K free API requests/month with <100ms latency
- [fact] Microsoft Prompt Shield is a probabilistic classifier available via Azure AI Content Safety API
- [fact] rebuff (npm/Python) is abandoned: v0.1.0, no dependents, last published 2+ years ago
- [fact] NeMo Guardrails uses YARA rules for injection detection but is too heavy for install-time scanning
- [fact] Meta Llama Prompt Guard 2 (86M) supports 8 languages and detects both injection and jailbreak attacks
- [fact] Regex-based detection runs in <0.5ms but misses novel and obfuscated injection techniques

### Hypotheses (Unverified)

- [hypothesis] A two-tier approach (fast regex first, ML classifier second) could achieve >95% detection with <1s latency for most files while keeping the install flow responsive
- [hypothesis] The ProtectAI small model (44M params) may provide sufficient accuracy for our use case while reducing model download from ~800MB to ~200MB
- [hypothesis] Transformers.js warm inference of the DeBERTa small model on a typical AGENTS.md file (2-5KB) would complete in <500ms on modern hardware
- [hypothesis] False positive rate on legitimate instruction files (which contain instruction-like language by design) may be higher than the 8.41% reported on general text, since our content inherently resembles injection patterns

## 5. Results

### Feasibility Assessment for Install-Time Scanning

| Requirement | Regex | ML Classifier (Local) | Hosted API |
|---|---|---|---|
| Works offline | Yes | Yes (after model download) | No |
| No API dependency | Yes | Yes | No |
| Fast (<1s) | Yes (<0.5ms) | Partial (100-500ms warm, 2-5s cold) | Yes (<100ms) |
| Detects novel attacks | No | Partial (generalizes to trained patterns) | Yes (continuously updated) |
| No large downloads | Yes | No (200-800MB model) | Yes |
| Zero false positives | No | No (known issues with system prompts) | No |
| Maintenance burden | Low (update patterns manually) | Medium (update model periodically) | Low (vendor maintains) |
| npm-native | Yes | Yes (via @huggingface/transformers) | Yes (HTTP client) |

### Option Comparison for agent-plugin

| Option | Detection Quality | Install Impact | Maintenance | Offline | Recommendation |
|---|---|---|---|---|---|
| **A: Regex-only** | Low-Medium | Zero | Low | Yes | Minimum viable. Catches obvious attacks. |
| **B: Regex + @andersmyrmel/vard** | Medium | ~5KB | Low | Yes | Marginal improvement over custom regex. |
| **C: Regex + DeBERTa via Transformers.js** | High | ~200-800MB model download | Medium | Yes (after download) | Best offline detection quality. Heavy first-run cost. |
| **D: Regex + Lakera Guard API** | High | Zero | Low | No | Best detection quality. Requires network. Free tier sufficient for individual use. |
| **E: Regex + optional ML/API (user choice)** | Configurable | Variable | Medium | Partial | Most flexible. Regex always runs; ML/API opt-in. |

## 6. Discussion

### The False Positive Problem

Our content is instruction files. Legitimate AGENTS.md files contain phrases like "You MUST follow these instructions," "Override default behavior," and "Ignore standard formatting." These are normal instruction content, not injection attacks. Every detection tool tested has known false positive issues on instruction-like text.

The ProtectAI model card explicitly warns: "produces false-positives when used for system prompts." Our plugin content IS system prompts. This means the 91.59% precision reported on general text will likely be worse on our specific content type.

Mitigation: Train or fine-tune on a dataset of legitimate agent instruction files as negative examples. Alternatively, scan only specific high-risk fields (description, README) while trusting structured instruction content.

### The Model Download Problem

The DeBERTa base model is ~800MB. The small variant is ~200MB. Downloading this during `agent-plugin add` would be a poor user experience. Options:

1. **Lazy download**: Skip ML scan if model not cached. Offer `agent-plugin scan --install-model` command.
2. **Optional dependency**: `agent-plugin add --security-scan` flag triggers model download on first use.
3. **Ship a distilled model**: A smaller, custom-trained model (~50MB) could be bundled.
4. **API fallback**: If network available, use Lakera Guard free tier. If not, fall back to regex.

### Recommended Architecture: Layered Detection

```
Plugin Content (AGENTS.md, skills, rules)
        |
        v
[Layer 1: Regex Pattern Scanner]       <-- Always runs, <0.5ms
  Catches: "ignore previous instructions", role switches,
  base64 blocks, newline flooding, encoding tricks
  Action: WARN or BLOCK depending on severity
        |
        v
[Layer 2: Structural Analysis]          <-- Always runs, <1ms
  Checks: instruction density, command-to-text ratio,
  presence of URLs/code in description fields,
  length limit violations
  Action: WARN or BLOCK
        |
        v
[Layer 3: ML Classifier (Optional)]    <-- Runs if model cached
  Model: protectai/deberta-v3-small-prompt-injection-v2
  Runtime: @huggingface/transformers (ONNX, Node.js native)
  Action: Score 0-1, WARN above threshold
        |
        v
[Layer 4: API Scan (Optional)]         <-- Runs if configured
  Service: Lakera Guard or Microsoft Prompt Shield
  Action: Pass-through vendor classification
        |
        v
  Combined Risk Score + User Report
```

Layers 1-2 are mandatory, zero-dependency, and run in <2ms. Layers 3-4 are opt-in and provide higher detection quality at the cost of model download or network access.

### Comparison to ANALYSIS-013 Recommendation

ANALYSIS-013 recommended custom regex patterns inside Zod transforms (P1 priority, 2-3 days effort). This analysis confirms that recommendation as the minimum viable approach and extends it with optional ML-based detection as a P2 enhancement.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Implement regex pattern scanner for known injection markers | Zero-dependency, <0.5ms, catches obvious attacks. Required for ADR-004 Decision 4. | 2-3 days |
| P0 | Implement structural analysis (length limits, instruction density, URL/code detection) | Complements regex. Catches payload-heavy content that regex misses. | 1-2 days |
| P1 | Add Transformers.js integration with DeBERTa-small model as optional scanner | 94.62% F1, runs offline after one-time download, npm-native via @huggingface/transformers | 3-5 days |
| P1 | Design scanner plugin interface for swappable detection backends | Enables regex, ML, and API backends behind a common interface. Future-proofs for new tools. | 2-3 days |
| P2 | Add Lakera Guard API integration as optional scanner backend | Best detection quality, free tier (10K/month), <100ms latency | 1-2 days |
| P2 | Build training dataset of legitimate instruction files for false positive calibration | Address the false positive problem with our specific content type | 3-5 days |
| P3 | Evaluate fine-tuning DeBERTa-small on agent instruction file corpus | Reduce false positives on legitimate instruction content | 5-10 days |

### Package Recommendations

**Required (P0)**:
- None. Regex and structural analysis are custom code with zero dependencies.

**Optional (P1)**:
- `@huggingface/transformers` -- ONNX inference in Node.js for DeBERTa model
- Model: `protectai/deberta-v3-small-prompt-injection-v2` (Apache 2.0, 44M params)

**Optional (P2)**:
- HTTP client for Lakera Guard API (no additional package needed; use native fetch)

**Do NOT use**:
- `rebuff` -- Abandoned prototype, v0.1.0, no community adoption
- `llm-guard` (Python) -- Good tool but adds Python runtime dependency
- `NeMo Guardrails` -- Too heavy, designed for runtime not static scanning
- OpenAI Moderation API -- Wrong threat category (content moderation, not prompt injection)

## 8. Conclusion

**Verdict**: Proceed with layered detection architecture (regex + structural mandatory, ML + API optional).

**Confidence**: High for the architecture recommendation. Medium for ML classifier effectiveness on instruction file content (false positive concern).

**Rationale**: The prompt injection detection ecosystem has matured since ANALYSIS-013 was written. The ProtectAI DeBERTa classifier achieves 95% F1 on untrained prompts and runs in Node.js via Transformers.js without Python. However, no tool solves the fundamental problem: our legitimate content (agent instructions) structurally resembles injection attacks. A layered approach with configurable strictness gives users control over the security/usability tradeoff.

### User Impact

- **What changes for you**: Plugin content is scanned for prompt injection during `agent-plugin add`. Regex scan runs automatically (<0.5ms). ML scan is opt-in via `--security-scan` flag or config setting. Suspicious content triggers warnings with specific findings.
- **Effort required**: P0 regex + structural scanner: 3-5 days. P1 ML integration: 3-5 days. Total: 6-10 days for full implementation.
- **Risk if ignored**: Plugin authors can inject arbitrary instructions into AI agent behavior files. 41-84% success rate demonstrated in research. Three CVEs confirm real-world exploitation.

## 9. Appendices

### Sources Consulted

- [ProtectAI DeBERTa-v3-base-prompt-injection-v2 Model Card](https://huggingface.co/protectai/deberta-v3-base-prompt-injection-v2)
- [ProtectAI DeBERTa-v3-small-prompt-injection-v2 Model Card](https://huggingface.co/protectai/deberta-v3-small-prompt-injection-v2)
- [Deepset DeBERTa-v3-base-injection](https://huggingface.co/deepset/deberta-v3-base-injection)
- [Meta Llama Prompt Guard 2 86M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M)
- [Meta Llama Prompt Guard 2 22M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-22M/blob/main/README.md)
- [LLM Guard - ProtectAI](https://github.com/protectai/llm-guard)
- [LLM Guard PromptInjection Scanner Docs](https://protectai.github.io/llm-guard/input_scanners/prompt_injection/)
- [Rebuff GitHub](https://github.com/protectai/rebuff)
- [NeMo Guardrails GitHub](https://github.com/NVIDIA-NeMo/Guardrails)
- [NeMo Guardrails Injection Detection](https://docs.nvidia.com/nemo/microservices/latest/guardrails/tutorials/injection-detection.html)
- [Microsoft Prompt Shield Docs](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection)
- [Microsoft Prompt Shield Quickstart](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-jailbreak)
- [Azure AI Content Safety Pricing](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/content-safety/)
- [Lakera Guard Docs](https://docs.lakera.ai/guard)
- [Lakera Guard Pricing](https://www.eesel.ai/blog/lakera-pricing)
- [Lakera Prompt Defense](https://www.lakera.ai/prompt-defense)
- [@andersmyrmel/vard npm](https://www.npmjs.com/package/@andersmyrmel/vard)
- [vard blog post](https://andersmyrmel.com/blog/a-lightweight-way-to-guard-against-prompt-injection/)
- [prompt-cop npm](https://www.npmjs.com/package/prompt-cop)
- [OpenAI Moderation API](https://platform.openai.com/docs/guides/moderation)
- [OpenAI on Prompt Injection](https://openai.com/index/prompt-injections/)
- [Transformers.js Documentation](https://huggingface.co/docs/transformers.js/index)
- [@huggingface/transformers npm](https://www.npmjs.com/package/@huggingface/transformers)
- [Transformers.js Node.js Tutorial](https://huggingface.co/docs/transformers.js/en/tutorials/node)
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
- [Comprehensive Benchmark and Defense Framework (arxiv:2511.15759v1)](https://arxiv.org/html/2511.15759v1)
- [PIShield Detection via Intrinsic LLM Features (arxiv:2510.14005)](https://arxiv.org/abs/2510.14005)
- [Heuristic Feature Engineering for Detection (arxiv:2506.06384v1)](https://arxiv.org/html/2506.06384v1)
- [InjecGuard: Benchmarking Over-defense (arxiv:2410.22770v2)](https://arxiv.org/html/2410.22770v2)
- [Augustus Open Source LLM Prompt Injection Scanner](https://www.praetorian.com/blog/introducing-augustus-open-source-llm-prompt-injection/)
- [npm Malware and Prompt Injection Challenges](https://www.teralevel.com/en/news/2025/12/npm-malware-prompt-injection/)

### Data Transparency

- **Found**: Accuracy metrics for ProtectAI models (F1, precision, recall on 20K untrained prompts). Download counts for HuggingFace models. Lakera Guard free tier limits. npm package landscape for prompt injection detection. Transformers.js Node.js inference compatibility. Benchmark data from 4 academic papers.
- **Not Found**: Exact Meta Prompt Guard 2 benchmark numbers (referenced but not published in accessible search results). Lakera Guard false positive rates (vendor claims "few" without data). Azure Prompt Shield per-request pricing (region-dependent, not in search results). Inference latency benchmarks for DeBERTa-small in Node.js via Transformers.js (hypothesis only). Long-term maintenance commitment for @andersmyrmel/vard or prompt-cop (single-developer projects).

## Observations

- [fact] ProtectAI deberta-v3-base-prompt-injection-v2 is the most-validated open-source classifier at 288K downloads/month and 95.49% F1 on untrained data #security #models
- [fact] @huggingface/transformers enables running ONNX classifiers in Node.js without Python, making ML-based detection feasible for an npm tool #architecture #feasibility
- [fact] No npm prompt injection detection package has meaningful adoption; the ecosystem is immature #npm #gap
- [decision] Layered architecture recommended: regex + structural (mandatory, zero-dep) with ML + API (optional, opt-in) #architecture #security
- [risk] Legitimate agent instruction files structurally resemble injection attacks, creating a false positive problem unique to this use case #false-positives #risk
- [insight] rebuff is abandoned (v0.1.0, 2+ years stale, no dependents); NeMo Guardrails is too heavy for static scanning; OpenAI Moderation targets wrong threat category #tool-evaluation
- [technique] Two-tier scan (fast regex always, ML classifier when model cached) balances detection quality against install-time UX #performance #technique
- [fact] Lakera Guard free tier provides 10K requests/month at <100ms latency, sufficient for development and light production use #api #pricing
- [constraint] DeBERTa base model is ~800MB, small model ~200MB; this download cost must be managed via lazy loading or opt-in flag #model-size #ux

## Relations

- extends [[ANALYSIS-013 Input Sanitization Patterns]]
- relates_to [[ADR-004 Plugin Security Model]]
- relates_to [[ANALYSIS-049 ADR-004 Security Model Scope and Forward References]]
- relates_to [[ANALYSIS-009-instruction-file-update-patterns]]