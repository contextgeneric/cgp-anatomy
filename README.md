# The Anatomy of Context-Generic Programming: Understanding Fission-Driven Development in Rust

## [Read the report](report.md)

## Overview

This repository contains the "source instruction" for how the report **The Anatomy of Context-Generic Programming** is co-authored by an LLM. It contains both the human-written original content, and the instructions and drafts written by the LLM to produce the final report.

Even though the LLM does a heavy amount of writing, the main structure and details of the report is drafted by a human. The AI drafts have also been thoroughly reviewed by a human, with manual corrections or amendment instructions to correct any mistakes.

## Authorship Process

This section briefly describes the process of authoring this report.

### Human-Written Draft

First, a human written draft was produced and saved in [`human-original`](./human-original/). This draft contains the key ideas to be conveyed in the report.

### AI Draft

The first AI draft is written in [`ai-draft`](./ai-draft/). We provide instructions to ask the LLM to produce a hyper-detailed report that goes into as much details as possible.

The AI draft starts with an executive summary and a table of content. Each chapter is then written with a separate prompt to make sure that the produced chapter can fit within the LLM's context window. The original instruction and summary is attached to each prompt to ensure that the LLM does not forget them.

Although the AI draft is very detailed, there is unavoidably a lot of repetition and rambling, since it is sort of instructed to do so.

### AI Review

After the first AI draft is written, we ask the LLM to read through the whole draft again, and produce a detailed review together with suggestions on how to improve the report.

The review is saved in [`ai-revision-1`](./ai-revision-1/review.md), and is used in the next step.

### AI Revision

We pass both the draft report and the review to the LLM, and ask it to produce a revised copy of the report. We first ask the LLM to write a new summary and table of content for the revised report.

Each chapter of the revised report is written in a separate prompt to work around the context window limit. For each prompt, the original report, review, and instruction are re-attached to ensure that they are not forgotten by the LLM. Additionally, we prompt the LLM to first write down all the action items for that chapter based on the review, together with a detailed outline before proceed to writing the revised chapter.

### Human-AI Revision

After the revised report is produced, the human author reads through the entire report and manually identify places that need to be revised. The human author then writes amendment instructions to ask the LLM to revise a specific section in the report, such as by using a better code example.

We also ask the LLM do another round of revision, but this time chapter by chapter without a full review of the report. After a few rounds of manual revision, the final report is written in [`report.md`](./report.md).

## Tips

Here are a few tips of how to write similar reports using LLM, or tips that how this report could have been better generated.

### LLMs are lazy by default

LLMs are fine tuned to write things as brief as possible, and use short styles such as point forms or tables. To produce a detailed report, we need to prompt the LLM to instead write as long as possible and write in full sentences.

So far, the prompt for detailed writings work best on Claude, while some other LLMs like ChatGPT still try to be lazy even if we prompt it to write longer. The best way to ask for more detailed writing is to add modifiers like "hyper detailed", "deep dive", and ask it to target audiences with PhD and research background.

Another trick to ask LLM to write longer is to ask them first write a detailed outline of what they are going to write about. Since outlines tend to be short, even lazy LLMs can write rather detailed outlines. And after the outlines are written, they are forced to follow the outline and write things in more details than they otherwise would.

It is also better to ask LLMs to be hyper verbose first, and then let them do another round of revision later to shorten it. We can think of the verbosity as a way of forcing the LLM to think out loud, and that could let them come up with more interesting content to be extracted later on. With the original content in long format, the LLM is also more tuned to preserve more details in their revision.

### Do not ask LLMs to reorganize content

It is best for the human to come up with the overall structure of the report, do not let the LLM to rearrange or merge the chapters to shorten the content. LLMs are not very good at determining which content is important, and which content can be omitted. When given the chance, they will remove the "soul" of the report and dumb everything down to something that they would have written lazily from scratch.

The best way to preserve content is to keep the human draft in sync with the latest chapter organization, and ask the LLM to not remove anything that was in the original draft. Even then, be very careful every time you ask the LLM to shorten the writings, because it will still try to remove things that you consider important.

### LLMs are not good at coming out code examples

In the human draft, we did not include detailed code examples for every major section. This result in the LLM coming up with subpar or non-sensible code examples when writing the chapters.

The main problem with this is that it is very hard to prompt the LLM to come up with better examples after the fact. Vague instructions like "use query database instead of http request as the example" don't really work in all cases. The best is still just write all code examples by hand, and ask the LLM to write the content based on the given examples.

### Manual context management is the most effective

The best way to let the LLM write highly detailed report is to not use agent mode, and instead manage the context window manually. This report is mainly written with Copilot in ask mode, with the LLM output copied manually.

By managing the context window manually, we can fit significantly more information in the context window than it usually can. Writing a single chapter in the report can easily consume the entire context window, especially if we also want to attach the earlier chapters to remind it of what it had already written.

In general, the more details we can fit into an LLM's context window, the better if can perform for its subsequent writing. Whenever possible, we try to always attach the full report and instructions in each prompt, so that they don't get cropped by the editor before being sent to the LLM.
