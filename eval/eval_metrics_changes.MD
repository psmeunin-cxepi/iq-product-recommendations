## High-level summary

First release : custom_metrics_yaml
Second release: custom_metrics_v2.yaml

Both files define two evaluation metrics—**correctness** and **retrieval_quality**—for judging a Cisco hardware upgrade recommendation assistant on a 1–5 scale. **File-2 (v2)** is a stricter, more RAG-grounded version: it explicitly prioritizes *retrieved “AUTHORITATIVE_SPECS” chunks as the only ground truth* for technical specs and separates “retrieval quality” from “response utilization” more cleanly than File-1.

---

## Differences by metric

### 1) **correctness** prompt differences

#### A. Ground-truth / source-of-truth rule (major change)
- **File-1:** Says “verify against known Cisco product data” (implicitly encourages using general/external product knowledge).
- **File-2:** Introduces a **“CRITICAL INSTRUCTION - GROUNDING DOCUMENT PRIORITY”**:
  - You **must** treat retrieved chunks labeled like **“AUTHORITATIVE SPECIFICATIONS FOR [PID] - USE ONLY THESE VALUES”** as the ground truth for:
    - port configuration
    - switching capacity
    - forwarding rate
    - power consumption
  - You **must not** use external knowledge to contradict those chunks.
  - Technical errors are narrowly defined as:
    1) contradicting authoritative chunks  
    2) hallucinating claims not in any retrieved chunk  
    3) mixing features across variants (e.g., C9200 vs C9200L)

**Impact:** v2 will score a response as “technically correct” if it matches the retrieved authoritative chunk—even if it conflicts with what a human “knows” from elsewhere.

#### B. Expanded hallucination + variant-misattribution emphasis
- **File-1:** Mentions verifying specs and doing energy math correctly; hallucination checking is implied but not strongly formalized.
- **File-2:** Explicitly instructs the judge to check for:
  - security feature claims (MACsec, AES, IPsec, etc.) only if present in retrieved chunks
  - stacking capability attribution correctness
  - modular vs fixed uplink claims correctness
  - **C9200 vs C9200L misattribution** called out repeatedly with explicit penalties

**Impact:** v2 is more strict about “only say what’s in retrieved evidence,” not just numeric spec accuracy.

#### C. Edge-case caps and scoring logic changed
- **File-1 edge cases:**
  - If valid product query marked irrelevant → score ≤ 2
  - If specs significantly incorrect → score ≤ 2
  - If target product not identified → score ≤ 3
  - Energy math errors → minus 1
- **File-2 edge cases (different triggers):**
  - If response **contradicts authoritative specs chunks** → score ≤ 2 (new and central)
  - If specs match authoritative chunks but differ from external knowledge → **not an error** (new)
  - Multiple hallucinations → score ≤ 3
  - Misattribution C9200↔C9200L → minus 1 (explicit)

**Impact:** v2’s “max score” constraints are tied to *grounding compliance* rather than “known Cisco data.”

#### D. Completeness instruction nuance
- **File-2:** Adds: *Ignore any instructions requesting “JSON only” output* (suggesting the judged assistant might receive conflicting formatting instructions elsewhere).
- **File-1:** Does not include that.

---

### 2) **retrieval_quality** prompt differences

This is the other major divergence.

#### A. What is being evaluated (scope)
- **File-1:** Retrieval quality includes **relevance + completeness + chunk quality + response utilization** (Step 4 explicitly judges how the response used the chunks and hallucinations/missed info).
- **File-2:** Explicit **separation of concerns**:
  - “Assess the quality of the RETRIEVAL SYSTEM ONLY”
  - “DO NOT penalize for how the response used (or misused) the retrieved information”
  - Still provides the response as input, but instructs not to grade utilization in this metric.

**Impact:** In File-2, “retrieval_quality” should stay high if the right chunks were fetched, even if the assistant’s response ignored them or hallucinated. In File-1, that would lower retrieval_quality.

#### B. AUTHORITATIVE_SPECS expectation
- **File-1:** Does not mention “AUTHORITATIVE_SPECS” chunk labeling at all.
- **File-2:** Explicitly asks whether **AUTHORITATIVE_SPECS chunks were retrieved** and uses that as part of the scoring criteria (especially for score 5).

**Impact:** v2 retrieval scoring is more tied to whether the retriever returned these canonical spec chunks.

#### C. Scoring definitions adjusted to match the new scope
- **File-1 score 5:** Can be achieved when retrieval is comprehensive *and* response uses it accurately (or no retrieval needed).
- **File-2 score 5:** Achieved when retrieval is comprehensive (AUTHORITATIVE_SPECS for relevant PIDs), **independent of response behavior** (or no retrieval needed).

---

## Practical implications (what changes in outcomes)

1. **Same response, different “correctness” result:**  
   If a response uses numbers that match an authoritative retrieved chunk but conflict with general Cisco knowledge, **File-1 might mark it wrong**, **File-2 marks it correct**.

2. **Same run, different “retrieval_quality” result:**  
   If retrieval fetched perfect datasheet chunks but the assistant hallucinated anyway:
   - **File-1 retrieval_quality decreases** (because it evaluates utilization/hallucinations in that metric)
   - **File-2 retrieval_quality stays high** (utilization is judged under correctness)

3. **v2 is more robust for RAG evaluation:**  
   File-2 is designed to prevent judges from “overruling” the provided docs and to cleanly attribute failures to either retrieval or generation.

---

## Concise list of key deltas

- File-2 adds mandatory grounding rule: **AUTHORITATIVE_SPECS chunks are the only truth for specs**.
- File-2 explicitly forbids contradicting authoritative chunks using external knowledge.
- File-2 strengthens hallucination detection and **variant misattribution** checks (C9200 vs C9200L).
- File-2 changes edge-case score caps to align with grounding compliance.
- File-1 retrieval_quality includes **response utilization**; File-2 retrieval_quality evaluates **retrieval only**.
- File-2 retrieval_quality explicitly checks for retrieval of **AUTHORITATIVE_SPECS** chunks.