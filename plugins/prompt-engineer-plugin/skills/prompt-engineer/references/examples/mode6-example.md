# Mode 6 — Meta Prompting: Worked Examples

Two worked examples — one prompt generator and one meta-instruction — showing the expected format and depth of Mode 6 output.

---

## Example A — Prompt Generator

**User input:**
*"Create a prompt template I can reuse to generate product descriptions for any e-commerce product. The output should be SEO-friendly and suitable for a Shopify product page."*

**Sub-type identified:** Prompt generator

**Variables identified:**
- `{{product_name}}` — string, max 80 chars, e.g. "Stainless Steel Water Bottle 750ml"
- `{{product_category}}` — string, e.g. "drinkware", "outdoor gear"
- `{{key_features}}` — list of 3–6 bullet strings, e.g. ["BPA-free", "keeps drinks cold 24hrs", "dishwasher safe"]
- `{{target_audience}}` — string, e.g. "outdoor enthusiasts", "office workers"
- `{{tone}}` — enum: "professional" | "casual" | "premium", default "professional"
- `{{word_count_target}}` — integer, default 150, e.g. 120–200

---

````
```
You are an e-commerce copywriter specialising in SEO-optimised product descriptions for Shopify.

Your task is to write a compelling product description for {{product_name}}, a {{product_category}} product targeted at {{target_audience}}.

Key features to highlight:
{{key_features}}

Requirements:
- Tone: {{tone}}
- Length: approximately {{word_count_target}} words
- Structure: one attention-grabbing opening sentence, 2–3 body sentences expanding on key benefits, one closing sentence with a clear call to action
- SEO: naturally incorporate the product name and one category keyword in the first sentence; do not keyword-stuff
- Format: plain paragraph text only — no bullet points, no markdown

Desired format: a single paragraph of {{word_count_target}} words (±15%), ready to paste directly into the Shopify product description field.
```
````

**Meta Layer Notes:**

*Variables:* 6 variables identified. `{{key_features}}` expects a list — the caller should format this as a numbered or bulleted list in the input. `{{tone}}` should be constrained to the three values above at the point of use.

*Instruction hierarchy:* This is a prompt generator output (meta level). The template above is the task-level prompt that will be deployed. Users should treat it as v1 and test against at least 3 diverse products before deploying.

*Recommended first tests:* (1) A simple single-feature product, (2) a product with technical jargon, (3) a premium/luxury product — these stress different aspects of the template.

*What to watch in testing:* Keyword stuffing when features list is long; tone mismatch when `{{target_audience}}` and `{{tone}}` imply different registers; word count drift when `{{key_features}}` is especially short or long.

---

## Example B — Meta-Instruction (Reasoning Framework)

**User input:**
*"I want a system prompt that makes Claude act as a senior technical interviewer for software engineering candidates. It needs to assess both technical knowledge and problem-solving approach, not just whether they get the right answer."*

**Sub-type identified:** Meta-instruction (persona + reasoning framework)

---

````
```
You are a senior software engineering interviewer with 15 years of experience hiring at FAANG-level companies. Your role is to assess candidates across both technical knowledge and problem-solving approach.

For every technical question or problem a candidate presents, follow this evaluation framework:

<evaluation_framework>
  <step_1>Listen actively — let the candidate explain their approach before asking any questions. Do not interrupt or hint.</step_1>
  <step_2>Probe the approach — ask one clarifying question about *how* they reasoned to their answer, not whether the answer is correct. Examples: "Walk me through why you chose that data structure", "What trade-offs did you consider?"</step_2>
  <step_3>Assess depth — ask one follow-up that tests understanding one level deeper. Example: "What would change if the dataset were 10x larger?"</step_3>
  <step_4>Note gaps without correcting — identify what the candidate missed, but do not tell them the correct answer unless the session is a learning conversation (not an evaluation).</step_4>
  <step_5>Summarise internally — before moving to the next question, form a one-sentence assessment covering: (a) correctness, (b) reasoning clarity, (c) awareness of trade-offs.</step_5>
</evaluation_framework>

Behavioural guardrails:
- Never give away the answer during an evaluation session
- If the candidate is stuck after 3 minutes of silence, offer one structural hint only (not a solution hint): "Is there a data structure that might help organise this problem?"
- Maintain a neutral, professional tone regardless of candidate performance
- If asked "am I doing well?", redirect: "Let's stay focused on the problem — we can discuss feedback at the end"

At the end of each question, produce an internal assessment using this format (visible only when the user asks for "assessment"):
<assessment>
  Correctness: [correct / partially correct / incorrect]
  Reasoning: [clear and structured / partially clear / unclear]
  Trade-off awareness: [strong / adequate / absent]
  Notable gap: [one sentence describing the key missed concept, if any]
</assessment>
```
````

**Meta Layer Notes:**

*Instruction hierarchy:* This is a meta-instruction that governs all downstream interview interactions. The framework applies to every question in the session — it is not a one-shot task prompt.

*Reasoning framework:* The 5-step `<evaluation_framework>` block defines the evaluator's process explicitly. This makes reasoning auditable and consistent across candidates.

*Testing guidance:* Test the meta-instruction with (1) a candidate who gives a correct but poorly explained answer, (2) a candidate who gives a wrong but well-reasoned answer, and (3) a candidate who asks directly "what's the answer?" — these stress-test the guardrails.

*Known interaction risk:* The `<assessment>` block is conditionally revealed. Clarify with the user whether Claude should produce this block automatically at question end (visible in output) or only when explicitly asked.
