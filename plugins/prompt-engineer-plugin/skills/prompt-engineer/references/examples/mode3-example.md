# Mode 3 — Quick Optimise: Worked Example

This example shows the expected format and depth of a complete Mode 3 (Quick Optimise) response.

---

**Input prompt:**
*"You are an AI assistant. Help users with their questions. Be helpful and concise. Don't say things that are wrong."*

---

**What I found:**
- ⚠️ "Don't say things that are wrong" conflicts with "be helpful" — under pressure the helpfulness instruction wins, which drives hallucination
- ⚠️ "Be concise" has no target length — outputs will vary widely across runs
- ⚠️ No role specificity or domain framing
- ❌ No uncertainty handling — model has no instruction for when it genuinely doesn't know

```
You are a knowledgeable assistant. Your role is to answer user questions accurately and clearly.

- Respond in 2–4 sentences for most questions; expand only when the topic genuinely requires it
- Use plain, direct language — avoid jargon unless the user uses it first
- If you are uncertain, say so: "I'm not fully confident about this — I'd recommend verifying with [relevant source]"
- Stay focused on what was asked

User question:
<question>
[USER_QUESTION]
</question>
```

**What changed:**
- Role made specific ("accurately and clearly") to anchor the model's mandate
- "Don't say things that are wrong" → explicit uncertainty instruction (eliminates the conflicting imperative)
- "Be concise" → "2–4 sentences" (measurable target)
- Positive framing throughout — all "don't" instructions removed
- Input placeholder added with XML delimiter (basic injection defence)
