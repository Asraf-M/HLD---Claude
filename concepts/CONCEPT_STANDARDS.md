# Concept Note Standard and Template (concepts folder)

## Purpose

Use this file as the default instruction format for writing any concept note in the concepts folder.

Goals:
- Keep notes interview-friendly and practical.
- Explain both theory and production tradeoffs.
- Show what breaks, then show the correct approach.
- Provide reusable structure so every concept file looks consistent.

---

## Writing Style Rules

1. Start with a real scenario, not abstract definitions.
2. Use simple language and short sections.
3. Prefer concrete numbers (QPS, latency, scale) where possible.
4. Always include at least one failure path.
5. Always include tradeoffs (pros and cons).
6. End with interview-ready summary answers.
7. Use ASCII characters only when possible for maximum readability.

---

## Required Section Order

Use this order unless the topic strongly requires changes.

1. The Scenario
2. Without This Approach (What Breaks)
3. Correct Approach
4. End-to-End Flow (Start to End)
5. Architecture Under Load
6. Detailed Tradeoff Analysis
7. Common Doubt (Q and A)
8. Key Insights
9. Interview Answer (Concise)

---

## Section Expectations

### 1) The Scenario
- Define scale and user action.
- Show why the problem exists.

### 2) Without This Approach (What Breaks)
- Show 2-3 common wrong/naive approaches.
- For each: explain failure mode and impact.

### 3) Correct Approach
- Present layered architecture.
- Explain why each layer exists.
- Include at least one API/data example.

### 4) End-to-End Flow (Start to End)
- Show numbered request lifecycle.
- Include write path and read path.
- Include async boundaries (queue/events/workers).

### 5) Architecture Under Load
- Show major components and traffic direction.
- Mention bottleneck points and control points.

### 6) Detailed Tradeoff Analysis
- Add comparison table with at least:
  - correctness
  - latency
  - throughput
  - complexity

### 7) Common Doubt (Q and A)
- Add one practical confusion point.
- Give clear decision rule.

### 8) Key Insights
- 4-6 bullets of "what to remember".

### 9) Interview Answer (Concise)
- 6-10 lines.
- Should be directly usable in interview.

---

## Reusable Markdown Template

```markdown
# <Concept Title>

## The Scenario
<real system context + scale>

---

## Without This Approach (What Breaks)

### Scenario 1: <naive approach>
<short example>

Outcome:
- <impact>
- <impact>

### Scenario 2: <naive approach>
<short example>

Outcome:
- <impact>
- <impact>

---

## Correct Approach

### Layer 1: <component>
<what it does and why>

### Layer 2: <component>
<what it does and why>

### Layer 3: <component>
<what it does and why>

---

## End-to-End Flow (Start to End)
1. <step>
2. <step>
3. <step>

---

## Architecture Under Load

<ascii architecture>

---

## Detailed Tradeoff Analysis

| Option | Correctness | Latency | Throughput | Complexity |
|--------|-------------|---------|-----------|------------|
| A      |             |         |            |            |
| B      |             |         |            |            |

---

## Common Doubt: <question>

Question:
<one common practical doubt>

Answer:
<clear recommendation + condition>

---

## Key Insights
1. <insight>
2. <insight>
3. <insight>
4. <insight>

---

## Interview Answer (Concise)
<short final answer>
```

---

## Quality Checklist (Must Pass)

Before finalizing any concept file:

1. Is there at least one concrete scale example?
2. Is there at least one failure path?
3. Are tradeoffs explicit, not implied?
4. Is source-of-truth boundary clearly stated?
5. Is consistency model clearly stated (strong/eventual)?
6. Is the interview answer concise and reusable?
7. Is formatting clean and consistent with this standard?

---

## Naming Convention for New Concept Files

Use one of:
- <Concept>.md
- <Concept> - Examples and Tradeoffs.md
- <Concept> - Full Flow and Tradeoffs.md

Keep titles explicit and searchable.

---

## Reference Example

See `1 Million Users Same Product Cart Scenario.md` in this folder for a complete example that follows this standard.
