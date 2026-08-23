# METHOD.md — How AI Was Used to Study for This Exam

### What this is

This repo is to share how AI was used, from scratch, to prepare for and pass the AWS DVA-C02 exam over roughly two months (April–May 2026). The Notion notes that this repo is built from are the output of that process. Questions and quizs are quite random and lots of follow up discussions are included so I don't paste them into this repo.

### The workflow, step by step

**1. A two-month study plan, generated from the exam guide.** [plan.md](plan.md)

Starting from the official DVA-C02 exam guide (domains, weights, topics), AI was used to generate a month-by-month study plan spanning the full prep window.

**2. The plan broken into 19 modules, distributed across a daily schedule.**

The monthly plan was decomposed into 19 topic modules (EC2/Storage, S3, VPC, Databases, Lambda, and so on), each assigned to specific days so the whole syllabus was covered on a fixed cadence rather than left open-ended.

**3. Study notes generated per module.** → [skills/01-notes-generation.md](skills/01-notes-generation.md)

For each module, AI generated the actual notes — the content later reorganized into this repo's `zh/` and `en/` files. This was the first of four distinct AI-assisted study "skills" used in the process.

**A recurring review step, outside the four skills.** After each module's notes were generated, they were pasted into a *separate, fresh conversation* and checked with a prompt along the lines of "where might this be wrong." This draws on the idea behind LLM-as-a-Judge — using an instance with no generation-time context to evaluate the output, rather than asking the same conversation that just wrote the notes to grade itself — though without the full methodology (no rubric, no scoring, no multiple judges). It wasn't one of the four skills; it was a fixed habit applied after every module, not a designed capability. It occasionally caught real issues, most often ones caused by the passage of time rather than errors in reasoning — for example, S3's single-object size limit had increased from 5 TB to 50 TB in late 2025, and the notes initially reflected the older figure carried over from training data.

**4. Short-answer questions generated from the notes, with follow-up and gap-checking.** → [skills/02-qa-followup.md](skills/02-qa-followup.md)

A second skill turned each module's notes into short-answer questions, with the ability to ask follow-ups and probe for gaps in understanding rather than just checking rote recall.

**5. Multiple-choice questions modeled on the real exam format.** → [skills/03-mcq-generation.md](skills/03-mcq-generation.md)

A third skill generated multiple-choice questions styled after actual DVA-C02 exam questions — same format, same style of scenario-based phrasing — to practice under conditions closer to the real test.

**6. Performance analysis to find and fill gaps.** → [skills/04-gap-analysis.md](skills/04-gap-analysis.md)

A fourth skill analyzed answers to the practice questions, identified which areas were weak, and generated supplementary material to close those specific gaps — rather than re-studying everything indiscriminately.

### What this file is for

The point of documenting this is that "used AI to study for a cert" can mean a lot of different things — pasting a syllabus into a chat once, or building an actual loop of plan → notes → practice → gap analysis → repeat. This was the latter. The repo's notes are one visible output of that loop; the loop itself is what this file is describing.

---