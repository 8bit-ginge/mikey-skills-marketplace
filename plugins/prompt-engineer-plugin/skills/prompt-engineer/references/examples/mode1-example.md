# Mode 1 — Quick Create: Worked Example

This example shows the expected format and depth of a complete Mode 1 (Quick Create) response.

---

**Input task:**
*"classify customer reviews as positive, negative, or neutral"*

---

````
```
You are a sentiment classification expert. Your task is to classify customer reviews into
one of three categories: positive, negative, or neutral.

Only assign one category per review. If a review expresses mixed sentiment, choose the
predominant one.

Review: ###
[INSERT REVIEW HERE]
###

Sentiment (respond with exactly one word — Positive, Negative, or Neutral):
```
````

*Techniques applied:*
- **Role assignment** — "sentiment classification expert" focuses the model's expertise on this specific task
- **Positive framing** — specifies what TO do ("choose the predominant one") rather than what not to do
- **Input delimiters** — `###` separates the review from the instruction, reducing injection risk
- **Output format specification** — "respond with exactly one word" eliminates ambiguity in the response

---

Offer to do a production-quality version: *"This is the quick version. Want a more thorough pass where I interview you for context and explain all the technique choices?"*
