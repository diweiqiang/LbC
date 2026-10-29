# Link before Compress (LbC): Storyline-Based Compression for Issue-Level Magazine Summarization

Official open-source repository for the paper:

> **Link before Compress: Storyline-Based Compression for Issue-Level Magazine Summarization**  
> Weiqiang Di$^{1,2,*}$, Xuejun Xiao$^3$, Wei Cheng$^4$  
> $^1$Intelligent Information Technology Research Center, Jingchu University of Technology  
> $^2$Jingmen Cryptometry Application Technology Research Center  
> $^3$School of Foreign Languages, Jingchu University of Technology  
> $^4$School of Mechanical Engineering, Lanzhou Petrochemical University of Vocational Technology  
> $*$Corresponding author: [weiqiangdi@gmail.com](mailto:weiqiangdi@gmail.com)

---

## 1. Overview

News magazines such as *The Economist* distribute related evidence across heterogeneous sections: a *Briefing* may explain a policy decision, a *Finance & economics* report its market consequences, and a regional dispatch its local implications. We define **issue-level summarization** as generating a concise, citation-bearing brief from the body articles of a complete magazine issue (~60 body articles, 100,000–150,000 tokens across >10 sections) under a fixed word budget ($L = 1,500 \pm 10\%$ words).

Existing long-input summarization strategies face a structural bottleneck on this task:
- **Compress-then-Merge pipelines** (e.g., Map-Reduce, Hierarchical Merging) summarize individual articles in isolation before merging them, discarding relational cues whose salience depends on another article (**merge-time loss**).
- **Long-Context single-pass generation** feeds the entire issue at once but suffers from attention dilution and positional bias across dozens of heterogeneous articles.

We propose **Link-before-Compress (LbC)**, a training-free method that changes the compression unit from a layout unit (individual article or section) to a semantic unit (**relation-defined storyline**):

1. **Step 1 — Hook Extraction (`Map`)**: Distill each body article into a compact structured **Hook Card** (50–100 tokens) enumerating exact entities, events, and core claims without narrative abstraction.
2. **Step 2 — Storyline Discovery (`Link`)**: Inspect all Hook Cards of an issue in a single pass to group related articles into **multi-article storylines** based on three strict criteria (*shared event/decision aspects*, *complementary evidence*, or *explicit contrast/conflict*). Overlapping membership is permitted, and unlinked articles remain eligible as **singletons**.
3. **Step 3 — Storyline Compression (`Compress`)**: Synthesize each storyline in a single call directly from the **full original source texts** of its member articles, producing a passage with sentence-level `[article_id]` citations.
4. **Step 4 — Issue-Level Composition (`Compose`)**: Compose the final issue brief within the global word budget $L = 1,500$ words by jointly weighing **section span**, **aggregate source length**, and **editorial prominence** (*Briefing*, cover story, section-opening lead), resolving redundancy across overlapping storylines and propagating source citations to every sentence.

---

## 2. Repository Contents

- [`PROMPTS_LbC.md`](PROMPTS_LbC.md): Complete, self-contained English prompt for running the 4-step **Link-before-Compress (LbC)** summarization workflow directly on a magazine issue in an LLM Agent environment.
- [`PROMPTS_Evaluation.md`](PROMPTS_Evaluation.md): Complete, self-contained English prompt for the **Held-Out Editorial Reference & Source-Typed ACU Evaluation Protocol** (Phase 1 reference ACU decomposition & three-way source-support classification; Phase 2 scoring of Cross-article ACU Coverage, Single-article ACU Coverage, Sentence-Level Citation Support, and Article Coverage).
- [`examples/sample_issue/`](examples/sample_issue/): Complete end-to-end walkthrough artifacts on an authentic issue of *The Economist* (August 1st 2026 issue, 61 body articles, 55,181 words):
  - `step1_hooks.json`: All 61 extracted structured Hook Cards preserving exact relational anchors.
  - `step2_storylines.json`: 10 discovered multi-article storylines (covering 32 articles) and 29 preserved singletons.
  - `step3_passages.md`: Full-text synthesized storyline and singleton passages with sentence-level citations.
  - `step4_issue_brief.md`: Final citation-bearing Issue Brief (1,619 words, strictly within $1,500 \pm 10\%$) with full `[aXX]` grounding and a 61-article reference key.

---

## 3. Quick Start (Agent Environment)

### 3.1 Generating an Issue Brief with LbC
Provide an issue of *The Economist* (PDF or parsed body articles) together with the prompt in [`PROMPTS_LbC.md`](PROMPTS_LbC.md) to your LLM Agent:

1. The agent indexes all body articles (`[a01]`, `[a02]`, ...) while withholding ***Leaders*** and ***The world this week***.
2. The agent executes **Step 1 (Hook Extraction)** $\rightarrow$ **Step 2 (Storyline Discovery)** $\rightarrow$ **Step 3 (Full-Text Storyline Compression)** $\rightarrow$ **Step 4 (Global Composition)**, persisting intermediate state to temporary files when supported by the workspace.
3. The agent outputs:
   - **Discovered Cross-Article Storylines Overview**
   - **Citation-Bearing Issue Brief (1,350–1,650 words)** with `[aXX]` citations at the end of every sentence
   - **Source Article Reference Key** mapping `[aXX]` to `[Section] — Headline`

### 3.2 Evaluating a Generated Issue Brief
Provide the generated Issue Brief, the issue's body articles, and the withheld reference sections (***Leaders*** and ***The world this week***) together with [`PROMPTS_Evaluation.md`](PROMPTS_Evaluation.md) to the evaluation judge model:

- The judge decomposes the withheld references into Atomic Content Units (ACUs), classifies each ACU against the body articles as `single-article`, `cross-article`, or `not_derivable` (excluded from denominators), and scores the brief on all four metrics.

---

## 4. Main Experimental Results

Evaluated across **60 held-out issues** of *The Economist* (April 2025 – June 2026) under a matched output budget ($L = 1,500 \pm 10\%$ words):

### 4.1 Main Comparison (Table 2)

| System | Cross-article ACU cov. (%) | Single-article ACU cov. (%) | Cit. support (%) | Article cov. (%) |
|---|:---:|:---:|:---:|:---:|
| *Long-context generation* | | | | |
| Long-context (full issue, single call) | 27.8 | 53.2 | 66.3 | 40.9 |
| Long-context + Chain-of-Density (5 rounds) | 32.2 | 58.4 | 59.7 | 51.2 |
| *Compress-then-merge* | | | | |
| Map-Reduce (article-level) | 23.6 | <u>62.5</u> | 81.3 | **77.8** |
| Hierarchical merging (article $\rightarrow$ section $\rightarrow$ issue) | 26.8 | 60.4 | 78.4 | 63.8 |
| Context-Aware Hierarchical Merging (CAHM) | <u>35.1</u> | **63.1** | <u>82.2</u> | <u>69.2</u> |
| *Sequential accumulation* | | | | |
| Chain-of-Agents (CoA) | 30.9 | 55.8 | 75.1 | 49.5 |
| **LbC (Link-before-Compress)** | **53.7** | 61.7 | **86.2** | 58.3 |

### 4.2 Controlled Ablation: Grouping Unit vs. Compression Input (Table 3)

| Variant | Group / Input | Cross (%) | Single (%) | Cit. (%) | Art. (%) |
|---|:---:|:---:|:---:|:---:|:---:|
| Map-Reduce | None / Summary | 23.6 | **62.5** | <u>81.3</u> | **77.8** |
| Hierarchical merging | Section / Summary | 26.8 | 60.4 | 78.4 | <u>63.8</u> |
| LbC (Summary-only) | Storyline / Summary | <u>36.4</u> | 54.7 | 79.3 | 53.1 |
| **LbC (Full)** | **Storyline / Full text** | **53.7** | <u>61.7</u> | **86.2** | 58.3 |
| $\Delta$ grouping (`Sum.-only` $-$ `MR`) | | +12.8 | $-$7.8 | $-$2.0 | $-$24.7 |
| $\Delta$ full text (`Full` $-$ `Sum.-only`) | | +17.3 | +7.0 | +6.9 | +5.2 |

---

## 5. Data Availability & Copyright Notice

In compliance with copyright protections, raw full-text articles of *The Economist* are not redistributed in this repository. Researchers with legitimate access to *The Economist* issues can directly apply [`PROMPTS_LbC.md`](PROMPTS_LbC.md) and [`PROMPTS_Evaluation.md`](PROMPTS_Evaluation.md) to reproduce the full generation and evaluation pipeline on any issue.

---

## 6. Citation

```bibtex
@article{di2026lbc,
  title   = {Link before Compress: Storyline-Based Compression for Issue-Level Magazine Summarization},
  author  = {Di, Weiqiang and Xiao, Xuejun and Cheng, Wei},
  year    = {2026}
}
```
