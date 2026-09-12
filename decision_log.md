# Decision Log

1. **Chose AppleSupport as the brand** — high volume (106k replies) with naturally bounded, distinct issue types (bugs, account, hardware, billing), making intent categories cleaner to define than a noisier brand.

2. **Subsampled to Apple-only conversations (~213k rows) rather than using the full 3M-row dataset** — the assignment explicitly allows and expects subsampling; no need to process irrelevant brands.

3. **Matched customer↔brand tweets via `in_response_to_tweet_id`, not by customer author ID** — an early version matched by customer ID and leaked in unrelated conversations with other brands (e.g. AdobeCare, AmazonHelp). Fixed by matching on the actual reply link.

4. **Used single-turn pairing (last customer message → Apple's reply), not full multi-turn thread reconstruction** — full thread context would be more accurate but was out of scope for the time available. Tradeoff: some "customer messages" are actually context-free fragments (e.g. "Yep", "It's 10.3.2!") missing earlier turns. Documented as a known limitation, not silently ignored.

5. **Stripped @mentions and URLs from text before intent definition and prompting** — raw text was noisy for both human reading (defining intents) and LLM classification.

6. **Defined intents by reading real sampled data, not from an external taxonomy (e.g. not using Banking77 directly)** — assignment explicitly asks for intents "you define from the data."

7. **Used prompt-based few-shot LLM classification instead of training a classifier** — no time to build/label enough data for supervised training; few-shot with explicit intent definitions was fast and scored 92.1% against hand-labeled ground truth.

8. **Classification temperature = 0** — determinism matters for reproducible, comparable metrics.

9. **Built the golden eval set by pre-filling with the classifier's own predictions, then correcting by hand** (rather than labeling 239 rows fully blind) — faster, and still produces genuine independent ground truth since final labels are human-corrected, not LLM-authored.

10. **Excluded reply-quality `bad` examples from the RAG knowledge base** — only grounding replies rated `good` (later widened via heuristic) are indexed, so the system doesn't learn to imitate brush-offs.

11. **Broadened the "brush-off" filter from `"dm us"` to a blanket `"dm"` match after discovering leakage** — narrow phrase matching missed variants like "in DM" / "to DM", which slipped into the retrieval knowledge base and caused the generator to imitate deflection behavior. Confirmed via manual inspection of retrieved matches before fixing.

12. **Sampled 3,000 of ~48k candidate resolutions for the vector database**, not all of them — sufficient retrieval coverage without excessive embedding time.

13. **Reply generation temperature = 0.3** — slight natural variation, but still controlled; not fully deterministic like classification, not freely creative either.

14. **Escalation policy defaults to "escalate when uncertain"** — hardware/safety issues and any reply-quality-`bad` precedent always escalate; simple how-to/bug questions with known-good precedent can auto-handle. Conservative bias is intentional and stated explicitly, not hidden.

15. **Ran a blind human-vs-judge agreement check (n=10) before trusting the LLM-judge's scores** — found only 20% exact agreement and a **negative** Spearman correlation (-0.46), revealing the judge has a systematic leniency bias and doesn't reliably rank replies the way a human does. Reported this honestly rather than presenting judge scores as validated ground truth.
