---
title: "Back to the Corpus"
date: 2026-02-21
categories:
  - projects
tags:
  - ocr
  - data-quality
  - fppc
animation: ocr
excerpt: "The FPPC corpus had bigger quality problems than I expected. After benchmarking OCR engines and re-extracting 2,867 documents, corpus usability went from 79.4% to 98.6%."
---

In my [last post](/projects/i-built-a-search-engine-kind-of/), I wrote about building a search app for the FPPC's ~14,000 advisory opinions. During testing, I discovered the corpus had bigger quality problems than I expected.

<div class="ocr-animation-wrapper">
{% include ocr-animation.html %}
</div>

There was far more garbled text than I anticipated. After some investigation it seems the issue is a combination of low quality OCR from the source (FPPC), low quality OCR of my own, and many 1990s PDFs have font-encoding corruption despite being "born digital." The bigger issue is my quality_score that Claude Code built for the extraction pipeline was designed more for efficiency in extraction rather than ensuring high quality text in production. So back to the corpus repo I went.

## The Quality Score Evolution

The v1 quality_score measured words per page, the ratio of alphabetic to printable characters, the presence of FPPC patterns (dates, "FPPC", section headers), and had a penalty for OCR garbage. All of these were stepwise functions and there was a practical cap of 0.80. I did a few rounds of calibration, using subagents to spot check extracted text against the raw PDFs, and the resulting v2 was really just v1 with pipeline bug fixes — a case-sensitive file extension check that had blocked ~4,000 documents from extraction, some boilerplate removal issues, things like that. The calibration confirmed that the bugs were real but the scoring algorithm itself was the deeper problem. The quality_score completely missed subtle character-level OCR corruption like "Califomia" or "Cornrnission" — structurally valid words, but wrong.

I had Claude Code come up with a new quality_score. This time the score was designed to use the full 0-1 range, to distinguish the good from the excellent documents, smooth piecewise-linear curves, and some new scoring components, the most important of which was a dictionary check (fraction of words found in a 73K-word English dictionary with some built in tolerance for the fact that FPPC opinions are likely to have words not included in the dictionary).

The process was a bit experimental. Claude Code would design a new quality_score and then check it by having subagents sample opinions across the new score spectrum, convert the associated raw PDFs to images and have the subagents transcribe the images, compare against the extracted text, and then independently score the extraction, giving us a subagent-generated score to compare against the quality_score.

After getting the v3 quality_score into what seems like a good spot we were able to see the extent of the problem a little better. Our "minor issues" threshold was about 0.80 and below and we had about 3,000 and change documents falling below that threshold.

## Deciding What to Re-Extract

To back up a bit, the original quality_score was scoped to making a judgement call on extraction escalation. The extraction pipeline first attempted native extraction, if below the quality threshold it would try OCR with PyMuPDF, and if still below threshold escalates to olmOCR-2 via DeepInfra API. This was more of a cost saving approach, getting as much extracted as I could for free, escalating to the API only when needed.

Now I realize that cost saving approach left me with a corpus with some big holes in quality from a user perspective. So I wanted to go straight to the API to get good quality extraction for the 0.80 and below documents. I accepted I needed to incur a bit more API cost to make up for the FPPC's low quality scans and embedded corruption. I'll have to do the work, and eat the cost, to fix the FPPC's mistakes. So it goes.

## Benchmarking OCR Engines

But before eating those costs I decided to do a test run of 18 sample PDFs across the corpus, with different quality_scores and from different eras. I'd test the extraction against the v3 quality_score using the three main OCR models (non-LLMs) I saw available on DeepInfra. I had originally chosen olmOCR-2 almost on a whim — it seemed good enough for my PDFs and AllenAI is a cool lab, so that's what I went with. I decided if I was going to eat some more cost, I should ensure the best results and the best quality per dollar. Here are those results:

| Metric          | Native | olmOCR-2 | PaddleOCR-VL | DeepSeek-OCR |
| --------------- | ------ | -------- | ------------ | ------------ |
| Avg v3 score    | 0.671  | 0.887    | 0.710        | 0.720        |
| Avg Δ vs native | —      | +0.217   | +0.039       | +0.050       |
| Avg dict miss % | —      | 5.0%     | 26.1%        | 21.4%        |
| Avg cost/doc    | —      | $0.0006  | $0.0003      | $0.0001      |
| Errors          | —      | 0        | 0            | 0            |

olmOCR-2 was the clear winner across every quality tier and every decade in the corpus, often by a wide margin. PaddleOCR-VL and DeepSeek-OCR actually made things worse in many cases — particularly on degraded documents from the 1990s and 2000s, where they'd frequently score below the native extraction. olmOCR-2 never regressed on a single document. That meant I didn't need to alter the pipeline at all, just force olmOCR-2 usage for all my below-threshold PDFs.

## Re-Extraction Results

Then I began the basically two day process (approximately 16 hours) of running all those through the DeepInfra olmOCR-2 API. The cost wasn't bad, working out to roughly four dollars. The improvement was pretty pronounced:

| Tier                     | Before | After | Change |
| ------------------------ | ------ | ----- | ------ |
| < 0.50 (broken)          | 338    | 9     | -329   |
| 0.50–0.70 (degraded)     | 1,000  | 11    | -989   |
| 0.70–0.80 (impaired)     | 1,559  | 172   | -1,387 |
| 0.80–0.90 (minor issues) | 2,878  | 3,908 | +1,030 |
| 0.90+ (clean)            | 8,321  | 9,996 | +1,675 |

2,844 documents improved out of 2,867 attempted — 99.2% — with zero errors and zero regressions. Corpus usability went from 79.4% to 98.6%.

I still have about 200 opinions below the 0.80 threshold. I might try to see if using an LLM as an OCR engine is worth the cost of getting those opinions above threshold. Or I might just live with the relatively small number of poor quality extractions.

## What I Learned

The bigger lessons for me are twofold. First, data cleanup is hard and consistently where I end up spending most of my time, in this app and others. That might be a function of the code generation abilities of LLMs — in the sense that maybe there was a time when building the webapp was the hard part, but that's not the case for me these days. In law we work with tons of unstructured data, and there is a lot of it spread out across the internet from different sources. The hardest thing I have found about being able to actually use all that data is getting it cleaned and into some semblance of structure.

Second, good enough for an extraction pipeline does not necessarily mean good enough for production. The quality_score that Claude Code initially built was aimed at being good enough for extraction escalation decisions, which was the immediate task. It would have been better if I had been more explicit about focusing on corpus quality from the jump or been more active in asking Claude Code about how the quality_score worked and pushed on the assumptions it was making. It's a useful lesson I'll carry to the next project.
