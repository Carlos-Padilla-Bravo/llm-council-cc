# LLM Council — a Claude Code skill

### 🇪🇸 **[Léeme en español →](./LEEME.md)**

Turn one hard decision into five independent analyses, a blind peer review with a forced ranking, and a single verdict that tells you where the advisors agree, where they clash, and what to do first.

```
Where the Council Agrees ····· what several lenses reached on their own
Where the Council Clashes ···· the disagreement, not smoothed over
Blind Spots Caught ·········· what only surfaced in peer review
Council Ranking ············· avg. rank per advisor, from blind review
The Recommendation ·········· a real answer, not "it depends"
The One Thing to Do First ··· one concrete step
```

Adapted from Andrej Karpathy's [LLM Council](https://github.com/karpathy/llm-council). Karpathy's version dispatches a query to several different models, has them rank each other anonymously, and lets a chairman compile the answer. This is that pipeline rebuilt to run entirely inside Claude Code, with one honest substitution — documented below.

**Claude Code only.** Not built or tested for claude.ai.

---

## What it does

```
   your question
        │
   Phase 0 · frame            reads CLAUDE.md, memory/, referenced files
        │                     → one neutral framed question
        ▼
   Phase 1 · five advisors    5 parallel agents, one message
        │                     Contrarian · First Principles · Expansionist
        │                     Outsider · Executor
        ▼
   Phase 2 · blind review     responses shuffled to Response A–E
        │                     5 parallel reviewers, each keeping its own lens
        │                     → per-response notes + blind spots + FINAL RANKING
        ▼
   Phase 3 · chairman         average rank per advisor
        │                     → agrees / clashes / blind spots / ranking
        │                       / recommendation / one first step
        ▼
   verdict in chat  +  council-transcripts/{topic}-{date}.md
        │
   Phase 5 · html             OPTIONAL, only if you ask for it
                              rebuilt from the transcript, any time later
```

The five lenses are not personas. They are chosen to create three standing tensions — Contrarian vs Expansionist (downside vs upside), First Principles vs Executor (rethink it vs ship it), with the Outsider in the middle catching the curse of knowledge.

Four advisors read your actual project files (`Read`, `Glob`, `Grep`) so the advice is about your situation, not a generic one. The Outsider deliberately reads nothing — its entire value is having no context.

**Web search is off by default.** It is the most expensive part of a run by a wide margin. Advisors may make one targeted search if the local context genuinely can't support a factual claim, and must say so when they do. To turn it on for a whole session, add `--research` or just ask for it.

---

## How it differs from Karpathy's LLM Council

| Stage | Karpathy's original | This skill |
|---|---|---|
| Diversity | 4 different models from 4 different labs (gpt-5.1, gemini-3-pro, claude-sonnet-4.5, grok-4) answering an identical prompt | 5 thinking lenses, all on the same model. Claude Code *can* assign a different model per subagent; using one is a deliberate choice, explained below |
| Stage 1 | Same query to every model, no persona | Framed question + workspace context to every advisor, each with an assigned angle and tool access |
| Stage 2 | Responses anonymized as `Response A`, `B`, `C`… (four labels, one per model), per-response evaluation, then a strict `FINAL RANKING:` block, parsed and averaged | Same format verbatim, including average-rank aggregation, over five labels. Two changes: reviewers keep their lens so five reviews don't collapse into one, and they rank only the four responses they did not write |
| Ranking criterion | "Accuracy and insight", per his README. His actual Stage 2 prompt states no criterion — it asks what each response does well and poorly, then demands the ranking | **Most changes the decision.** Deliberately one-sided advisors can't be compared on answer quality |
| Stage 3 | Chairman model synthesizes freely | Main agent synthesizes into five fixed sections aimed at a decision, and may side with the dissenter over the majority |
| Context | Query only | Phase 0 reads `CLAUDE.md`, `memory/`, referenced files, past transcripts. No equivalent in the original |
| Output | Web UI with per-stage tabs | Verdict in chat + a full markdown transcript on disk, plus an HTML report on request |

### Why the differences exist

Karpathy's council rests on one property that does all the quiet work: **the judges are effectively blind.** Four models from four different labs answer the same prompt, and when they rank each other as `Response A` through `D`, none can reliably tell which answer is its own or who wrote the rest — different training, different house style, nothing to recognize. Every other rule in his design is safe *because* that holds. (Strictly, this is the design's premise rather than a measured fact; he does not test it, and neither has this repository.)

Claude Code can assign a different model to each subagent, so a literal port is technically possible. It is not what this skill does, for two reasons. Every model available in the harness comes from one lab, so mixing them buys tier variation, not the cross-lab independence Karpathy's design is built on — the shared priors stay shared. And mixing tiers actively corrupts the ranking: a Haiku advisor placed last tells you about model capability, not about whether its angle was worth hearing. So the diversity here is manufactured through assigned angles, on equal footing.

That choice has a consequence: an angle is self-identifying. The Contrarian is recognizable from its first sentence, to a reader and to itself. Copy Karpathy's rules unchanged onto that foundation and you inherit the shape of the method without the thing that made it work. Three places where that matters, and what this skill does instead:

**1. Reviewers must not share one prompt.** In the original, the four reviews differ because four different models wrote them — the prompt is identical for all of them, and that is fine. Give five instances of *one* model that same identical prompt and you get five near-identical reviews: five agents spent to buy one opinion. Here each reviewer keeps its own lens while judging, so the reviews diverge for the same reason the answers did.

**2. "Best answer" is not a rankable question here.** Karpathy's models each give their honest best attempt, so ranking them on accuracy and insight is coherent. These advisors are instructed to be one-sided; asking which is "best" compares a deliberate pessimist to a deliberate optimist. The criterion becomes **which response would most change the decision of a smart person facing this problem** — a question that stays meaningful when the inputs are biased by design.

**3. A reviewer cannot rank itself.** In the original, every model ranks every response, its own included. Worth being exact here, because his README and his code disagree: the README says each LLM "is given the responses of the *other* LLMs", but `stage2_collect_rankings` in `backend/council.py` builds one prompt containing all of Stage 1 and sends that same prompt to every council model. So each one does see and rank its own answer. It is harmless there — a model has no way to pick its own out of four strangers.

Ours can pick it out instantly. In testing, a reviewer placed its own answer first despite an explicit instruction not to favor its own angle; it read the rule, and the pull of agreeing with itself won. Restating the prohibition more forcefully is the fix that feels right and changes nothing — the model already understood. So reviewers here rank only the four responses they did not write.

**Honest about that fix: it works, but not every time.** In measured runs roughly one reviewer in five still slips its own answer into the ranking, usually at the top. Phase 3 therefore checks compliance before aggregating, discards a non-compliant ranking, states how many were counted, and — when the result is close — recomputes with the discarded ranking repaired to confirm the order does not depend on that call.

Each of those is a departure from the letter of the method in order to preserve what the method depends on. They are documented rather than hidden because a reader deserves to know which decisions are Karpathy's and which are this fork's.

**What the substitution still costs:** five instances of one model share priors and blind spots that four independent labs would not. Convergence across the lenses is a strong hypothesis, never independent proof — and no amount of prompt engineering fixes that, nor would swapping in a second model from the same lab. If you need genuinely independent judgment, run the original against four APIs.

---

## When to use it

Good council questions:

- "Should I launch a $97 workshop or a $497 course?"
- "Which of these three positioning angles is strongest?"
- "I'm thinking of pivoting from X to Y. Am I crazy?"
- "Here's my landing page copy. What's weak?"
- "Should I hire a VA or build the automation first?"

Bad council questions:

- "What's the capital of France?" — one right answer
- "Write me a tweet" — creation task, not a decision
- "Summarize this article" — processing task, not judgment

The council earns its cost when there is genuine uncertainty and being wrong is expensive. If you already know the answer and want validation, it will probably tell you the thing you were avoiding. That is the point.

---

## Install

Clone into your user skills directory to have it everywhere:

```bash
git clone https://github.com/Carlos-Padilla-Bravo/llm-council-cc.git ~/.claude/skills/llm-council
```

```powershell
git clone https://github.com/Carlos-Padilla-Bravo/llm-council-cc.git "$env:USERPROFILE\.claude\skills\llm-council"
```

Or drop it in one project only, at `.claude/skills/llm-council/` in the repo root.

The folder holds `SKILL.md` (the skill itself), `report-template.html` (used only by the optional HTML report), plus this README and the license, which Claude Code ignores.

Verify: start Claude Code and type `/llm-council`.

---

## Usage

Any of these triggers it:

- `council this: <your question>`
- `run the council on <your question>`
- `pressure-test this`
- `stress-test this`
- `war room this`

It also fires on a genuine tradeoff phrased naturally — "should I X or Y", "I'm torn between", "which option". It will not fire on lookups, writing tasks, or a casual "should I" with nothing at stake.

Abridged output:

```markdown
## Council Verdict: beginner course pricing

### Where the Council Agrees
The beginner solopreneur angle has real demand — but the framing sells a tool,
not an outcome, and non-technical buyers won't recognize the tool.

### Where the Council Clashes
Price. The Contrarian says $297 is too high against free content; the
Expansionist says it's too low for what a bundled community is worth.

### Blind Spots the Council Caught
Three of five reviewers landed on the same omission: nobody costed support
load for a non-technical cohort.

### Council Ranking
| Advisor | Avg rank | Rankings counted |
|---|---|---|
| The Outsider | 1.6 | 5 |
| The Executor | 2.2 | 5 |
| ... | | |

The Outsider ranked first because every other advisor silently assumed the
audience already knows the product name.

### The Recommendation
Don't build the course. Validate with a lower-commitment offer and reframe
around the outcome.

### The One Thing to Do First
Run one $97 live workshop titled for the result, not the software.
```

---

## Output

The verdict is delivered in the chat. A full transcript is always written to `council-transcripts/{topic-slug}-{YYYY-MM-DD}.md` in your working directory: the framed question, the context files consulted, all five full advisor responses, all five full reviews, the letter→advisor map, the ranking table, and the verdict.

The council reads that folder on later runs, so it won't re-argue ground you already settled.

Output language mirrors yours — ask in Spanish, get the verdict in Spanish. The skill itself is written in English.

### HTML report (optional)

Ask for it — "make an HTML report", "dame el informe en HTML" — and you get a self-contained page next to the transcript. It never runs on its own, because most verdicts are read once and acted on.

You can request it days later in a different session: it is rebuilt from the markdown transcript, so the council never has to reconvene. The markdown stays the source of truth; the HTML is a view of it.

**Styling.** `report-template.html` ships in a dark-first tech register — near-black canvas, monospace metadata, electric accent, hairline rules — with a light-mode variant that follows the reader's system setting. It carries no personal or company branding, and every color, font, and spacing value is a CSS custom property in the first 40 lines. Two ways to make it yours:

- Edit those custom properties once, in your own copy. Nothing else in the file needs to change.
- Or, if you have a brand-identity skill installed, the report picks it up automatically and overrides those properties with your tokens.

The template itself must stay neutral — it is shared by everyone who installs the skill. If you fork this repo and hardcode your brand into it, you'll break that for anyone downstream.

---

## Cost & caveats

- **~10 agents per run.** Five advisors, five reviewers, chairman inline. That is the design, not an accident. Don't run it on questions that don't deserve it.
- **Web search dominates the cost** when enabled, which is why it is opt-in rather than default. For scale: in one measured run with search off, each advisor consumed roughly 23-25k tokens; searching multiplies that by an amount this repository has not measured. The optional HTML report, by contrast, is one template read and one file write.
- **The panel is author-built.** These five lenses were chosen by a human. Agreement between them is a strong hypothesis, never a consensus of the field.
- **Anonymization is partial.** Fixed lenses are self-identifying. Read the ranking as a rough signal, not a scoreboard.
- **Average rank is not a vote.** A lens can finish last and still carry the observation that decides the question. The chairman is instructed to say so when it happens, and to side with a dissenter whose reasoning is strongest.

---

## Credits

- **Method:** [Andrej Karpathy — llm-council](https://github.com/karpathy/llm-council). The three-stage structure, the anonymized peer review, the strict `FINAL RANKING:` format and the average-rank aggregation are his. That repository carried **no license** when this one was written (checked 2 August 2026), so nothing is claimed about its terms here — and nothing needed to be, because no code or text from it is reproduced. What is reused is the method, which the README and `backend/council.py` describe openly.
- **Prior adaptation:** the idea of porting the council to a Claude skill with five thinking lenses is credited to [Ole Lehmann](https://x.com/olelehmann). It circulates in at least two near-identical distributions — [tenfoldmarc/llm-council-skill](https://github.com/tenfoldmarc/llm-council-skill) and [aiwithremy/claude-skills-llm-council](https://github.com/aiwithremy/claude-skills-llm-council), whose `SKILL.md` files are 96% identical sentence-for-sentence — and neither carries a license. This repository shares **no text with either** — measured, zero identical sentences — and reworks the design: reviewers keep their lens, Karpathy's ranking is restored, the chairman is the main agent, advisors read project files, web search is opt-in, and the output is a chat verdict plus a transcript.
- **This implementation:** [Carlos Padilla Bravo](https://github.com/Carlos-Padilla-Bravo).

The contents of this repository are MIT licensed. See [LICENSE](./LICENSE).
