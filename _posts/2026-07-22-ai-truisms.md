---
title: "What's True about AI"
date: 2026-07-22
tags: AI, development
---

I used to love programming. I still do, but I do it a lot less now.

Things feel different. Everybody is talking about AI, and there's a cloud that hangs over the field of software development: Are our jobs going away? Are we no longer differentiated by the things we spent so long learning? Do Boris Cherney, Pete Steinberger, and all the other AI thought-leaders know something we don't? Or are these people just living in an alternate reality where tokens are free and everything AI does it lovely?

I wanted to write down a simple list of ideas that have been running around in my head. These are things which I believe are facts, but are at least true for my lived experiences.

## Humans are sometimes bad at code
I've read and written plenty of bad code. Poor abstractions, unnecessary defensiveness, re-inventing the wheel, security vulnerabilities: you name it, I've probably done it.

I've read code and not understood it. I've read code and not realized that I didn't understand it until much later.

## AI is sometimes bad at code
All of the same things humans do, AI does. It does it faster, requires an external LLM provider, and costs money per turn.

## Agentic development feels very different [Consider removing]
I'm not sure if it's different and it sucks, or it's just different and I don't like change.

## Writing instructions in English is hard
Using English to prompt a stochastic genie to model business goals feels like trying to add pepper to a pot of soup, except I'm standing two stories above the kitchen. Sometimes I add too little, sometimes I add too much, sometimes I totally miss my target.

## I can't read all of this
Ask the model something. Terminal window fills up several times over. Scroll up. Read a little bit. Feel overwhelmed and just accept that the model is probably right.

Models are confident,overly verbose, and love jargon. I have observed this internal defense mechanism: "I don't understand this. I must be too stupid to understand this. I don't want to be or appear stupid, so by agreeing, I can bypass that."  This internal conflict leads to more slop than anything else.

Bonus points: humans can do this too, I just hadn't noticed it so starkly before. When I hear someone say "domain," I immediately have no idea what they're talking about. Is this a bounded context? Is this a feature of the product? Is this simply an area of code? [Nobody knows](https://www.youtube.com/watch?v=JYqfVE-fykk).

## I am suspicious of all human writing
I don't know if people were always saying "it's not X, it's Y," but I'm hyper-aware of it. When I see it, I assume it was written by AI. I am starting to look for the fingerprints of LLM generated text everywhere. It's exhausting to be suspicious all the time.

## Using AI to write messages for humans sucks
If you couldn't be bothered to write the message yourself, it signals to me that you don't care about me or what you're saying.

I'm tired of docslop. There's so much reading we're expected to do now; the cost of writing giant documents in Google Docs, Linear, JIRA, or Notion has gone to zero. I'm dubious that people who generate giant documents read every line of them.

I don't believe that people receiving the documents are reading the documents either, I think they're asking their agent to summarize it. The pipeline of written communication is **person A -> person A's agent -> person B's agent -> person B**.

## Differing outcomes [EXPAND]
The blind men touching the elephant describing it totally different 

## Different feature areas feel totally different
Feature areas within an existing codebase have [differing risk tolerances](https://www.youtube.com/watch?v=DZpR0GojoWQ). Using AI to write or modify code that touches payment processing? sweating bullets. Using AI to add a brand new feature or product? Fuck it, we ball.

People having such vastly different experiences with agentic develop, either pro or con, are likely the result of the above. Additionally, codebases with:
* low cohesion
* high coupling
* poor or inconsistent architecture
* mounds of outstanding tech debt
* conflicting agent markdown files
* poor test coverage
* hinge on undocumented/tribal knowledge
* written in languages less represented in the training set

will naturally have a vastly different agentic development experience than a codebase which is the opposite.

## Adversarial code reviews are powerful
Whether I write the code myself or an agent does, having two separate models review the code has been a win, hands down. Remember, both AI and humans are sometimes bad at code. [Multi-model code reviews](https://www.skills.sh/cosmastech/skills/multi-model-code-review) add more Swiss cheese to the process.

## AI doesn't learn
This is painful. Every time an agent goes off the rails, we want to add a new skill file or modify our CLAUDE.md or AGENTS.md, ending up with [scar-tissue](https://medium.com/machine-words/ptsd-post-traumatic-software-design-51e7eac44664) markdown that eats tokens and probably just confuses the LLM.

Either that or I'm continuously writing in my prompt what I want the agent to avoid.

## AI doesn’t understand
Conman is short for "confidence man," who abuse their victim's trust by projecting confidence. AI is a token generation process. In my experience, LLMs don't stop and ask me to clarify when they don't understand what I'm asking. Skill issue.

AI doesn't understand business context unless it's provided. Business context is usually very deep, scattered in many places, and it's [hard to recognize what's our implicit understanding](https://en.wikipedia.org/wiki/Curse_of_knowledge) that we don't put into context.

## AI isn’t responsible for failures
AI can aid in detecting root cause of incidents and failures, but there's no firing the agent if it was their code which caused the incident. They are like the owner's son: no matter what they do, they ain't getting fired. In fact, they'll probably continue to get promoted.

## Deep human understanding is irreplaceable
I've been in the weird position where I started a new job right around the time that agentic development became the norm. The code looks like what you might expect a startup's codebase to look like with 7 years of development and a hundred or more devs working on it.

Trying to grok a codebase with years of tribal knowledge, varying engineering practices, latent bugs, and complex async processes is hard. Harder still when the expectation is to be extra productive with the aid of AI.

The people who understand the code deeply are able to glance at a planning document or a bug ticket and provide more steering than I may ever be able to. The code is not a part of my DNA, nor is it part of the LLM's DNA: at the end of each session, the context has to be rebuilt.

## Context is the current war
Don't give too little context. Don't give too much context. Don't let the LLM get into the dumb zone by going past the 30% of the context window. Summarize. Don't summarize, start a new session instead. Use hand-off docs. Don't use hand-off docs.

## The value of agentic first development hasn't materialized
Each new state of the art model's cost-per-token exceeds the previous version substantially. The chickens will have to come home to roost eventually: be it layoffs or spending caps.

### Saying yes is easier, even when saying no is probably better
It's easier than ever to say "yes" to whatever your customers want. Saying "no" is psychologically harder than saying "yes." AI allows us to avoid saying "no, we won't support that," at the cost of complexity for engineers, AI agents, users, and support.

![Just one more setting for a single client bro](/assets/2026/just-one-more-setting-bro.jpg)

### Is our product better?
It's really easy to build a new feature and add it to your product. I don't believe it's materially easier to know whether or not that feature is wanted by users or will be used.

## Vanity refactors are easier than ever
For the low cost of whatever my employer is paying for tokens, I can refactor code so it meets my personal sensibilities. :sunglasses: Just pay that cost again when someone else has different sensibilities.

## Thrashing and continuous decision making sucks
I have three or four agent sessions running right now. All of them are waiting on me to provide input. I think back to [Alan Watts' little parable about the farm laborer](https://www.youtube.com/watch?v=YTCyloOL2DY) who gets promoted. He goes from chopping logs and mending fences to deciding which potatoes should be kept or thrown away. The laborer quits after one day because continuous decision making is too much for him.

It's hard to feel productive while an agent is doing it's work and it's very tempting to open a new session to make it seem like I'm doing something. More decisions, more context switching, less flow.

## Software development doesn't feel the same
The sort of person who liked software development five years ago received a different set of rewards in the process of building things. The tension of "I don't understand this problem" and the resolution of "I figured it out" are either gone or materially shifted into something unrecognizable to my dopamine receptors.

It stands to reason that the rewards of the new software development process would not feel (as) good to the profile of a person who liked pre-agentic development.
