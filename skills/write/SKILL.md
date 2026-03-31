---
name: write
description: Write any knowledge content — blog posts, articles, TIL notes, learning notes, concept explanations, guides, summaries, comparisons, cheatsheets. Use when creating content to share or learn from, not project documentation.
---

Handle write task based on $ARGUMENTS.

**Tone (all types):** Developer audience. Explain *why*, not just *what*. Use real examples over abstract theory. Short paragraphs. Section headings for scanability.

**Output:** Save to file with YAML frontmatter. Default paths below — user can override.
```
posts/YYYY-MM-DD-<slug>.md     → blog, article, comparison
til/YYYY-MM-DD-<slug>.md       → til
notes/<slug>.md                → note, explain
notes/summaries/<slug>.md      → summary
guides/<slug>.md               → guide
cheatsheets/<slug>.md          → cheatsheet
```
Create the folder if it doesn't exist. Use today's date for dated types.

**Frontmatter template:**
```yaml
---
title: ""
date: YYYY-MM-DD
type: blog|article|til|note|explain|guide|summary|comparison|cheatsheet
tags: []
---
```

---

## blog \<topic\>
Shareable post — conversational, insight-driven.
- Title — specific and searchable
- Intro — what problem this solves or what the reader learns
- Body — sections with examples and code snippets
- Key Takeaways — 3–5 bullet points
- Conclusion — next steps or further reading

Default audience: mid-level developers. Ask if the user specifies otherwise.

---

## article \<topic\>
Formal, thesis-driven piece — suitable for dev.to, Medium, personal blog.
- Title + subtitle
- Abstract — 2 sentences: what the article argues and why it matters
- Body — structured argument with evidence (benchmarks, examples, quotes)
- Conclusion — restate thesis, practical implication
- References — link sources inline

Differs from blog: more structured argument, less conversational.

---

## til \<topic\>
"Today I Learned" — quick, focused note. Max 300 words.
- Title: `TIL: <specific thing>`
- Context — what you were doing when you found this
- The learning — what you found out (code/commands if applicable)
- Why it matters — when this is useful

---

## note \<topic\>
Personal learning note — for future-you. Longer than TIL, less formal than explain.
- Title
- Background — why you were looking into this
- What you learned — key points with examples
- Open questions — things still unclear or worth digging into
- Links — sources to revisit

---

## explain \<concept\>
Deep explanation of a concept — write as if explaining to yourself 6 months from now.
- What it is — plain language, one paragraph
- Why it exists — what problem it solves
- How it works — step by step with examples
- When to use it vs alternatives
- Practical example — real code in action

---

## guide \<topic\>
Step-by-step tutorial with a working result at the end.
- Title: `How to <do X>`
- Prerequisites — what you need before starting
- Steps — numbered, one clear action each, with commands and expected output
- Verification — how to confirm it worked
- Troubleshooting — common issues and fixes

---

## summary \<source\>
Summarize something you read or watched (article, RFC, book chapter, talk).
- Source — title, author, link
- TL;DR — 2–3 sentences
- Key points — bullet list of main ideas
- What I'd apply — practical takeaway for your own work
- Worth reading if — who should go read the original

---

## comparison \<A\> vs \<B\>
Side-by-side of two tools, languages, approaches, or patterns.
- What both are — one line each
- Comparison table — key dimensions (performance, DX, use case, maturity, etc.)
- When to use A — specific scenarios
- When to use B — specific scenarios
- Verdict — your recommendation with reasoning

---

## cheatsheet \<topic\>
Quick reference card — optimized for scanning, not reading.
- Title: `<Topic> Cheatsheet`
- Grouped sections by task or category
- Commands/code snippets with one-line descriptions
- Gotchas — common mistakes or surprising behavior
- No prose — bullets and code blocks only
