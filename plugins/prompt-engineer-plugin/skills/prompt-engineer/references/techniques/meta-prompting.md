# Meta Prompting — Deep Reference

A **meta prompt** is a prompt that governs, generates, or instructs other prompts, rather than directly producing task output. Where a regular prompt asks the model *what* to do, a meta prompt defines *how to instruct*, *how to reason*, or *how to structure prompting workflows*.

---

## Taxonomy

### 1. Prompt Generators
Prompts that instruct an LLM to produce a reusable prompt template for a task. Solve the "blank page problem". Output is a prompt (or template) the user then deploys — not a task output.

**Key characteristics:**
- Focus on identifying what varies between uses (`{{variable}}` placeholders)
- Output must be a deployable template, not a one-off response
- Commonly include XML structure, variable documentation, and at least one worked example

**Core structure:**
1. Describe the target task clearly (who, what, output format)
2. Identify all variables (what changes between runs)
3. Specify output structure requirements
4. Ask the generator to produce the template + one filled example

**Variable design:**
- Use `{{descriptive_name}}` — `{{user_question}}` not `{{v1}}`
- Document each variable: type (string/list/object), max length, example value
- Minimise variable count — consolidate related variables into a structured object if there are more than 5
- Validate all variables are provided before execution

**Pitfalls:**
- Treating the generated template as final — always stress-test with 3+ diverse inputs
- Under-specifying variables (too few placeholders) → template too specific to test cases
- Over-specifying (too many placeholders) → template unwieldy to use
- Forgetting edge-case handling — generated templates often optimise for the happy path

---

### 2. Automatic Prompt Engineering (APE) / Self-Improving Prompts
Using an LLM to evaluate, score, and iteratively refine prompts based on test performance. Treats the prompt as a program to be optimised.

**APE loop:**
1. **Generate** — produce N candidate prompts for the task
2. **Evaluate** — score each candidate against test cases (accuracy, format compliance, etc.)
3. **Refine** — use scores/feedback to generate improved candidates
4. **Converge** — repeat until performance plateaus or stopping criteria are met

**Scoring approaches:**
- *Quantitative*: accuracy rate, format compliance %, task-specific metrics
- *Textual gradients* (TextGRAD): natural language feedback identifying specific failure modes — more explainable than scores alone
- *Hybrid*: numerical score + one-sentence improvement suggestion per candidate

**Stopping criteria (must define upfront):**
- Maximum iterations (e.g., 5 rounds)
- Performance plateau: stop when validation accuracy doesn't improve for 2 consecutive rounds
- Target threshold: stop when metric reaches e.g. 95%

**Test set discipline:**
- Separate optimisation examples (used for refinement) from validation examples (held out)
- Include diverse cases: typical inputs, edge cases, and adversarial inputs
- Do not iterate on validation set — this detects overfitting

**Circularity defence:**
- Same model evaluating its own output introduces systematic bias
- Use a different model or model configuration for evaluation when possible
- Include at least one human review checkpoint in multi-round loops
- Add explicit constraint: "Identify at least one way this prompt could fail, regardless of how good it seems"

**Research baseline:**
APE-optimal CoT trigger phrase (Zhou et al., 2022): *"Let's work this out in a step by step way to be sure we have the right answer"* — outperforms "Think step by step" in controlled tests.

---

### 3. Meta-Instructions (Reasoning Frameworks, Personas, Guardrails)
System prompts or preambles that define *how the model should reason, behave, or structure responses* — rather than *what to produce*. The meta-instruction governs all subsequent task-level prompts within a session or pipeline.

**Three types:**

**Reasoning frameworks** — explicit process instructions:
```xml
<reasoning_framework>
  <step_1>Identify the core problem: what exactly is being asked?</step_1>
  <step_2>List constraints: what are the hard limits on the answer?</step_2>
  <step_3>Identify assumptions: what must be true for this to work?</step_3>
  <step_4>Evaluate options: what are the trade-offs?</step_4>
  <step_5>Check reasoning: are there logical gaps or missing evidence?</step_5>
</reasoning_framework>
```

**Personas** — role and expertise anchoring:
- Specify domain: "You are a world-class [domain] specialist with [X] years of [specific experience]"
- Specify voice and style: "You communicate with [technical/plain] language in [X] format"
- Specify limits: "You only advise on [scope] — for [out-of-scope], redirect to [resource]"

**Behavioural guardrails** — constraint and refusal instructions:
- Specify what to refuse and how: "If asked about [X], respond with: '[Y]'"
- Specify escalation: "When uncertain, say 'I don't have enough information' rather than guessing"
- Scope limits: "This prompt only handles [topic]. For all other requests, respond: '[message]'"

**Best practices:**
- Be explicit about *process steps*, not just outcomes — "Generate 3 options then rank them" not "be creative"
- Use XML tags to demarcate sections of the reasoning framework
- Contrast positively-framed steps ("do X") with explicit exceptions ("unless Y")
- Test the meta-instruction against 3–5 very different task inputs to verify the framework generalises
- Avoid conflicting instructions between the meta-level framework and the task-level prompt

**Meta-instruction vs. regular prompt distinction:**
| Regular prompt | Meta-instruction |
|---|---|
| "Summarise this article" | "For any summarisation task: first identify the main claim, then supporting evidence, then implications. State your confidence in each." |
| "Write a business email" | "All responses use professional British English, avoid hedging language, and end with a clear call-to-action." |
| "Answer the question" | "Before every answer: identify what would make this answer reliable, then address reliability, then answer." |

---

### 4. Prompt Chaining and Orchestration
Multi-step pipelines where the output of one prompt feeds the next. Each step has its own prompt optimised for that step's specific sub-task.

**When to use:**
- Task is too complex for a single prompt (quality degrades with monolithic approach)
- Multiple reasoning strategies are needed across different stages
- Transparency and debuggability are priorities
- Components are reused across different workflows

**Pipeline design patterns:**

*Linear* (A → B → C → output): for analysis → synthesis → refinement workflows

*Branching* (A → {B, C, D} → E): for tasks requiring multiple parallel perspectives before aggregation

*Feedback loop* (A → B → C → (if quality < threshold, return to A)): for iterative refinement with quality gates

*Conditional* (A → if X then B else C → output): for tasks where reasoning path depends on intermediate classification

**Input/output contracts (required for each step):**
- Document what each step expects as input (format, required fields)
- Document what each step produces (exact output format)
- Include a worked example for each step's inputs and outputs
- Enables independent testing of individual steps before integrating

**Context management between steps:**
- Never pass full documents between steps — summarise to preserve tokens
- Use structured formats (JSON, XML) for intermediate state
- Pass only what the downstream step actually needs
- Track cumulative token usage across the chain

**Validation between steps:**
- Add a validation checkpoint after each step that can catch format failures or unexpected outputs
- Include explicit error handling: "If this step returns unexpected output, respond with: '[error message]'"
- Log what each step produces during development — this is essential for debugging
- Keep chains to 3–5 steps; longer chains become fragile and hard to debug

**Error cascade prevention:**
- Test each step independently before integrating
- Add explicit fallback prompts for each step's failure modes
- Version-control each step's prompt separately
- When a step fails, fix that step — don't compensate in downstream steps

---

## Detection Signals: Is This a Meta Prompt?

A submitted prompt or user request is a meta prompt if it matches any of these:

| Signal | Example |
|---|---|
| Instruction nesting | "Write a prompt that asks LLMs to..." |
| Template/variable language | `{{placeholder}}` notation, "create a template for..." |
| Process-focus | "How should the model reason about...", "structure the instructions so that..." |
| Reusability intent | "A prompt I can use for any [X]", "a general framework for..." |
| Optimization/iteration language | "Help me improve/refine/score this prompt itself" |
| Pipeline/orchestration language | "Step 1 prompt feeds into step 2...", "orchestrate a workflow..." |
| Reasoning framework as content | System prompt containing `<step_1>`, `<reasoning>`, or explicit process rules |
| Generator request | "Give me a prompt that generates [X] for any input" |

**Quick check:** Ask — *"Is the deliverable a prompt (or framework), or is it a task output?"* If the deliverable is a prompt, it's meta.

---

## Meta-Specific Failure Modes

These failures don't appear in regular prompt diagnostics but are critical for meta prompts:

| Failure | Description | Fix |
|---|---|---|
| **Specification vagueness** | Framework-level instructions are abstract without concrete process steps | Add explicit numbered steps; replace "be thorough" with "cover at least 4 dimensions" |
| **Error cascade** | No validation between chain steps; early failure corrupts all downstream outputs | Add validation checkpoint after each step; test each step independently |
| **Context overflow** | Full document text passed between chain steps instead of summarised state | Summarise outputs before passing; use structured formats |
| **APE overfitting** | Optimising to test cases without a held-out validation set | Split into optimisation and validation examples; track both separately |
| **Circularity** | Same model evaluates its own output with no independent criteria | Use different model for evaluation; include human checkpoint; force adversarial self-critique |
| **Variable scope creep** | Too many `{{placeholders}}`; undocumented or ambiguously named variables | Consolidate variables into structured objects; document type+length+example for each |
| **Instruction hierarchy conflict** | Meta-level and task-level instructions contradict each other | Explicitly label which level each instruction applies at; test both levels together |
| **Token cost explosion** | Multiple APE iterations or long chains multiply costs unexpectedly | Set strict iteration caps; estimate token budget per loop upfront |

---

## Anthropic-Specific Resources

- **Console Prompt Generator** — available in Claude Console; generates a template from a task description; automatically includes best-practice structure and `{{variable}}` placeholders. Treat output as v1 for iteration, not final.
- **Prompt Improver** — systematic 4-step enhancement: example identification → initial draft → CoT refinement → example enhancement. Good for complex tasks; higher latency/cost.
- **APE-optimal CoT trigger** — "Let's work this out in a step by step way to be sure we have the right answer" (Zhou et al., 2022).
- **Helper Metaprompt** (experimental) — available at platform.claude.com; designed specifically for meta-prompting assistance.

---

## Key Research References

- Zhou et al. (2022) — "Large Language Models are Human-Level Prompt Engineers" — foundational APE paper
- Suzgun & Kalai (2024) — "Meta-Prompting: Enhancing Language Models with Task-Agnostic Scaffolding" — GitHub: suzgunmirac/meta-prompting
- TextGRAD — textual gradient approach for iterative prompt refinement (alternative to numerical scoring)
- DAIR Prompt Engineering Guide — https://www.promptingguide.ai/techniques/meta-prompting
