# Citation Check

> No citation → no output.

Citation Check is a small, opinionated module for **detecting and blocking hallucinated or missing citations** in LLM outputs.

Built for research, slides, and agent pipelines where correctness matters.

---

## Checked when its all gud
https://github.com/user-attachments/assets/da72770c-85dc-4c8c-b3f4-1cb5e28cadfb
## Identified Issues
<img width="768" height="429" alt="Screenshot 2026-01-22 at 12 37 02" src="https://github.com/user-attachments/assets/d600af61-8edc-4f94-b721-b375cb1a8c39" />
<img width="749" height="602" alt="Screenshot 2026-01-22 at 12 37 24" src="https://github.com/user-attachments/assets/0fa1863a-5188-4ac9-865f-429cd1ae755e" />

## What it does

- checks that every factual claim has a citation  
- verifies that cited sources actually exist  
- optionally validates claim–citation consistency  

If a check fails, the output should be blocked or regenerated.

---

## Why

Most tools add citations *after* generation.  
This enforces citations as a **generation constraint**.

If your agent can’t cite a claim, it shouldn’t make it.

---

## Typical flow

```text
LLM output
↓
Citation Check
↓
pass → ship
fail → regenerate / stop
