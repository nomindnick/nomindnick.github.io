---
title: "I Built a Search Engine. Kind Of."
date: 2026-02-18
categories:
  - projects
tags:
  - search
  - claude-code
  - fppc
animation: search
excerpt: "I recently built a search engine for California FPPC advisory opinions. It works well — in my testing, it's significantly better than what the FPPC has on their website. But if you asked me to explain how it works, I'd struggle."
---

I recently built a search engine for California FPPC advisory opinions. It works well — in my testing, it's significantly better than what the FPPC has on their website. But if you asked me to explain how it works, I'd struggle. This was the first project where I don't really understand a big part of what I built. Claude Code built it, Claude Code tested it, and Claude Code decided when it was done. I think I'm ok with that?

<div class="search-animation-wrapper">
{% include search-animation.html %}
</div>

Let me back up.

The FPPC is California's agency that enforces conflict of interest, lobbying, and campaign finance laws. They also issue advisory opinions on request from public officials. Their website has about 14,000 of these opinions going back to the 1970s, all as PDFs, with a search function that has frustrated me for years: straight keyword matching, no way to see at a glance which results are actually relevant, forcing you to open each PDF individually.

I thought building an alternative search would be fairly straightforward. I've built RAG systems before, so this would be easier because I'm not adding an AI layer. It's just an app. The data exists, I just need to scrape it, do a bit of cleaning, tack on a search engine, and throw together a front end. The thing I didn't know is that adding a search engine is complicated. What I also didn't know is I wouldn't need to figure it out.

I have been "vibe coding" since early 2024, back before that was a term and the process was copying and pasting individual functions or maybe files back and forth between an AI chatbot and your IDE. I have also been trying to learn some python on the side. The old vibe coding process had some advantages in terms of learning, because it forced me to get into the code. I had to structure the repo, I had to manage the files, I had to find the right function in the code and copy and paste it back and forth, and occasionally I'd fix something obvious that I recognized to save myself the copy-pasting effort. That forced me to understand, at least at an architectural level, what I was building, even if I didn't fully understand how the code actually worked.

I have been using Claude Code since it was released a year ago, which is obviously a different flow, but it still kept me in the loop enough to give me a sense of control and understanding of what was being built. Most often, I would need to guide Claude Code, relying mostly on intuition and what little I understood about architecture. I would have kind of a vision for how a system would work, Claude Code would build per that vision, and maybe it would work, often not, but Claude Code would give me the feedback and guidance to help me find the right design, and I'd learn a little along the way, even if I was no longer even reading much of the code.

What I found interesting about building the search engine was just how out of the loop I felt. Claude Code built the testing infrastructure, Claude Code built search engines to test, Claude Code would run the experiments, investigate the results, and determine what the next experiment should be. I had, and have, very little understanding of the search engines that "we" built and tested. Claude decided when we had found the best solution we were going to get and it was time to stop experimenting and ship. Claude was making choices and decisions.

In early testing, I am impressed with the results. The final search engine is orders of magnitude better than the FPPC website, so goal accomplished. I don't exactly know how I feel about letting myself get cut out of the loop to this extent. But the experimental approach worked, and I think it's something I will continue to expand on for complex projects.

## The Corpus

For this project, I broke things into separate repos. This is not something I've done before, but I anticipated I might need to do some trial and error experimentation on the search engine and wanted to keep things separated.

I started with a repo just to build the scraper.

The only other scraping I'd done was pulling bills off the California legislature's website, which was trivial — uniform URLs, identical page layouts. The FPPC opinions are all PDFs with no uniformity among file names, so this was a different challenge.

My plan was to talk to Claude Code, explore the website, test different ways to scrape, and then once we landed on an approach we'd run the full scrape. But Claude Code just kind of did some initial exploring and was like, "Hey, I figured it out. Want me to just do it?" And then it just...did it. My plan of going back and forth with Claude Code over a few days to figure out a scraping plan, and learning a bit along the way, was out the window. The scrape was done before it started.

From there we continued with processing. Claude Code built a processing pipeline to pull the text from each opinion, pull out legal citations, citations to other opinions, dates, categories (conflict of interest, lobbying, etc.) and other metadata into a structured JSON file, with one file for each opinion. A big challenge was just getting good text out of the PDFs, particularly for opinions from the 1970s and 1980s.

Of particular interest, modern opinions start with "Question" and "Answer" headings, which are high level summaries of the question presented to the FPPC and the FPPC's ultimate conclusion, then followed by the full factual background and analysis. I knew I wanted these broken out in the JSON as 1) those headings could be useful for search, and 2) displaying those for the user in the webapp would make it easier to quickly scan results for what is relevant. The problem is that format was not consistently followed throughout the decades. Opinions from the past 20 or so years are easy to pull out the Question/Answer sections with regex. But older opinions have inconsistent formats or no headings like that at all (instead just diving into the analysis). I overcame that by running each opinion that we couldn't extract Question/Answer sections from through Claude Haiku to have it generate the Question/Answer blocks. I had Claude Code use subagents to spot check a few hundred of these to ensure the Question/Answer that was generated was accurate.

The other interesting thing is the processing pipeline pulled citations to other FPPC opinions from each opinion. This allows for the creation of a citation graph and the ability to follow how specific topics are analyzed over time. This is a common feature of legal research tools, such as Westlaw, so definitely a bonus to get that here and something clearly lacking from the FPPC's website.

## The Evaluation Problem

With the corpus set up, I created a separate repo to build a scoring system for different search techniques. We created 65 different queries, and used sub-agents to explore the corpus via grep/keyword searching to identify the most relevant opinions in response to those queries. That became the basis for scoring: does the search engine return the opinions that the sub-agents found? In the right ranked order? Claude Code's scoring system returns a number of metrics that I admittedly don't fully understand, despite Claude Code explaining them to me a few times. But the core concept is simple enough: did the engine find the right opinions, and did it rank them well?

If I were being a rigorous data professional, I'd have built a golden set by hand — searching the corpus myself and identifying the relevant opinions for each query. But this is an evenings-and-weekends side project, and I wasn't going to spend my free time researching made-up conflict of interest hypotheticals. So I had Claude Code spin up sub-agents to do it. I don't think I lost much. The first-order task is keyword searching through files, and I'm probably worse at exhaustive keyword searches than the agents are. I might do a better job assessing the relevance of any given opinion with my domain expertise, but on the task of actually finding candidates the agents are probably more capable than me. If there was a keyword search class in law school, I must have missed it.

## Nine Experiments

For the search engine itself, I put the scorer and the corpus into a new repo, set it up to be experimental, and started with BM25 as a baseline. I still don't quite understand what BM25 is, other than fancy keyword search. Gun to my head, I couldn't tell you what it does differently from what the FPPC already has on their website.

From there, the approach was to iterate through different experiments to see how the scores improved over the baseline. My instinct from previous RAG work was that embeddings would improve results. So that was the next experiment.

The experimental setup had Claude Code generate a research report after each experiment: what was done, how the current test performed versus the baseline, what likely caused any regressions or improvements, and suggestions for other experiments to try. It also reported out the actual results for each of the 65 queries, so we could see at the per-query level which queries struggled, which individual opinions were getting missed, and how they were ranking.

Embeddings performed worse than BM25, which surprised me. Then Claude started using the word "fusion" and we went experimenting from there. We ran nine main experiments total.

My approach, which wasn't planned in advance, was to split the work across two Claude instances. Claude Code ran the experiments: building each search engine variant, running it against the evaluation suite, comparing the granular output files from the current and past experiments, and generating detailed result files. Meanwhile, I'd take those result files to Claude on the web, which served as my analyst. I gave it the README from the experiment repo and the schema for the JSON files, and after each experiment I'd share the results. It would analyze the scores, identify which queries regressed and why, and suggest what to try next.

This turned out to be a surprisingly effective workflow. Claude Code was doing the engineering, Claude on the web was doing the analysis, and I was the project manager in between. Project manager might be a charitable framing, perhaps I was more the useless middle manager shuffling messages between the actually knowledgeable workers. Claude on the web very patiently tried to explain to me what was happening along the way, what BM25 is, what fusion means, and all of that. I was admittedly more nodding along than really understanding.

We ended up at experiment nine with something that scored, in Claude's opinion, probably the best we could get without doing more advanced and expensive techniques like query expansion or cross-encoder re-ranking. For this application, I'm not trying to introduce expensive API calls. I want this thing to basically run for free. We did one final experiment to see if we could fine-tune the search engine further. It didn't improve results, so we let it be.

## What I Ended Up With

My basic understanding of the final search engine is: there are two paths. If the user enters a statute as part of their search query, we use the citation graph to pool candidates and we use embeddings for that. Otherwise, with no statutes detected, it's just BM25.

Which is kind of funny. We ran nine experiments trying to beat BM25, and for most queries, BM25 won. The citation pooling and embeddings path matters — it rescues the queries that BM25 can't handle, particularly when someone searches for a specific statute number. But for the majority of searches, the baseline we were trying to improve against ended up being the thing we mostly shipped. Sometimes the simplest approach is the right one, and the value of all that experimentation was proving it rather than finding something exotic.

## What I'm Taking Away

This was the first time I've built something where I don't understand a big part of the architecture. I am putting a lot more trust in Claude than I normally would. There are two things that make that feel manageable: 1) the search engine works, and is giving me good results in my testing, and 2) multiple Claude instances have been assessing this and giving me the same explanations, whether or not I actually understood the explanations.

The experimental approach worked really well. Between Claude Code running the experiments and Claude on the web analyzing the results, things moved forward systematically and I could feel confident in the results. That's an approach I'll take forward for other complex projects. And now that it's done, I kind of want to go back and actually learn what I built, have Claude walk me through the repo, explain the different techniques, what worked and what didn't. I noted throughout this that I didn't understand what was being built, but that was more of a choice on my part. Claude frequently explained things to me (I have "explanatory" style turned on in Claude Code specifically so I can try to learn along the way) but I wasn't taking the time to really dig in.

My other takeaway is simpler: building a search engine is harder than I thought. I came into this thinking, "Oh yeah, just strap a search engine on." It is always fun to learn how little I know about this stuff.

But that's the "kind of" in the title. I built a search engine in the sense that it exists, it works, and it solves the problem I set out to solve. I didn't build it in the sense that I couldn't rebuild it from scratch or even explain its architecture in any detail. Usually I could say I built it in the sense that I at least had the initial vision for the architecture or pipeline, but as to the search engine itself that is not the case here. But I did build it in the sense that I created the conditions that made it a reality. The project was my idea, I setup the repos and the experimental structure for Claude Code to figure out the best search engine for my corpus, and I kept the project moving along the way. Whether these distinctions matter probably depends on what you think "building" means in 2026. I'm still figuring that out.
