# AI Support Agent for AppleSupport — Report

## 1. Problem framing

**What "good" means for this brand:** Apple's Twitter support handles a high volume of repetitive bug/update complaints, alongside a smaller but higher-stakes tail of account-access, billing, and hardware-safety issues. For this brand, "good" means:
- Reliably triaging the high-volume, low-risk bulk (bug reports, how-to questions) so humans aren't burning time on repetitive tickets.
- Being conservative — not confidently automating anything involving money, account access, or physical safety, where a wrong answer causes real harm or liability.
- Replies that reference a specific, real fix rather than generic reassurance — customers in this dataset are frequently venting about *already* having tried generic troubleshooting.

**What we chose not to build:**
- Full multi-turn conversation modeling (see Decision #4) — single-turn pairing was used instead, given the time budget.
- A fine-tuned classifier — few-shot prompting was faster to build and validate within 2 days, and performed well (92.1%).
- Sarcasm/tone detection as a separate signal — several messages in the data are sarcastic ("thanks apple 🙄") and would confuse a naive classifier; out of scope here, noted as a failure mode below.
- Production concerns (logging, monitoring, rate-limit-aware queuing, cost tracking) — this is a prototype proving the approach works, not a deployable service.

## 2. Results vs. baselines

| Approach | Intent accuracy |
|---|---|
| Trivial (always guess majority class) | 55.2% |
| Simple (keyword matching) | 41.8% |
| **Our LLM classifier (few-shot)** | **92.1%** |

Note: the simple keyword baseline scored *below* the trivial baseline. The dataset is heavily skewed (`bug_performance_issue` = 55% of all labels), so naive keyword rules can be confidently wrong more often than a baseline that does nothing intelligent at all. The real lift to report is the ~37-point gain over the *best* baseline (55.2%), not over 0%.

**Reply quality** (from 239 hand-labeled real Apple replies, used to build the grounding knowledge base): 75 good / 157 partial / 7 bad.

**Escalation split** (golden set, human-labeled): 185 escalate / 54 auto — reflecting the conservative-by-design policy (Decision #14).

## 3. Failure analysis — top 5 modes

1. **Context-free fragments from single-turn pairing.** Messages like "It's 10.3.2!" or "Thanks but as mentioned, it only saves it as an image when this is done" are mid-conversation replies missing earlier turns. The classifier and generator both received these with no way to recover the missing context, occasionally producing plausible-sounding but ungrounded output. *Root cause: Decision #4 (single-turn pairing).*

2. **Ambiguous intent boundaries.** E.g. "my phone acting up, never dropped it" straddles `bug_performance_issue` and `general_complaint` — no named symptom, but implies a bug. These cases don't have one "correct" label; they're a genuine property of the taxonomy, not a bug in the classifier.

3. **RAG knowledge base leakage before filtering.** An early version of the grounding knowledge base still contained "let's take this to DM" brush-offs because the filter only matched the exact phrase `"dm us"`, missing variants. The generator dutifully imitated this pattern until the filter was broadened. Fixed, but illustrates how retrieval quality is only as good as the filtering logic behind it.

4. **Generated replies occasionally exceed the specificity of their grounding examples.** In one case, the model produced a detailed numbered how-to (Print → PDF export steps) that was more specific than any of its three retrieved examples — meaning it blended retrieved context with its own general knowledge. This is a hallucination risk: the output *looks* grounded but isn't fully verifiable against real precedent.

5. **LLM-judge shows a leniency bias and doesn't reliably rank reply quality.** See section 4 below — this is significant enough to also be its own finding.

## 4. What is misleading about my headline number

The 92.1% classifier accuracy and the LLM-judge's average quality scores are the two "headline" numbers this system could be sold on. Both need caveats:

- **92.1% accuracy is measured against a 6-way taxonomy with skewed class distribution** — the majority class alone is 55.2% of the data (the trivial baseline). 92.1% is a real, meaningful lift, but should always be read against that 55.2% floor, not a 0% floor.
- **The LLM-judge's quality scores are not reliable ground truth.** A blind human check against 10 of the judge's scores found only 20% exact agreement, an average absolute difference of 1.10 (on a 1-5 scale), and a **Spearman correlation of -0.46** — meaning the judge doesn't just miscalibrate the *number*, it disagrees with a human about *which* replies are even better than others. The judge consistently scored higher than the human rater in 6 of 10 cases. Any "average reply quality" number derived from this judge should be treated as optimistic and unvalidated, not as ground truth. (Caveat: n=10 is small; this is a warning sign, not a precise estimate.)

## 5. What we'd do next with one more week

- Rebuild the human-judge validation with a much larger sample (50-100 rows) to get a stable agreement estimate, and experiment with a **pairwise comparison judge** ("which of these two replies is better?") instead of absolute 1-5 scoring — pairwise judgments are generally more reliable for LLMs than absolute scales.
- Reconstruct full multi-turn threads instead of single-turn pairing, to fix the context-fragment failure mode.
- Add a sarcasm/tone-mismatch detector as a pre-filter, since sarcastic "thanks" messages currently risk being misread as positive or neutral.
- Fact-check generated replies against an actual Apple support knowledge source (rather than only tweet-derived precedent) to reduce the hallucination risk identified in failure mode #4.
- Expand the golden eval set beyond 239 examples and stratify sampling explicitly by intent, to get more stable per-intent accuracy numbers (currently `hardware_safety_issue` has only 6 examples).
