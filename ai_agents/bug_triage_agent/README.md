# Morpheus - AI bug triage agent

An AI-powered bug triage agent that transforms raw, messy bug reports into decision-ready intelligence briefs for software engineering teams.

## The Problem

Engineers waste 40–60% of their debugging time before they ever write a single line of fix — just gathering context, guessing severity, and figuring out who owns the issue. On a typical team with 10+ open bugs at any given time, that's hundreds of hours per quarter lost to triage overhead that feels productive but isn't. Every bug that lands in Slack or GitHub as a vague paragraph forces a senior engineer to stop what they're doing and play detective.

## The Solution

Morpheus runs a 5-stage agentic pipeline using the Claude API to analyze any raw bug report and produce a structured Bug Intelligence Card in under 60 seconds. The user pastes a bug report (freeform text, stack trace, or both) into the terminal. If critical context is missing, Morpheus asks the 2–3 targeted follow-up questions a senior engineer would ask — then proceeds to diagnose severity, hypothesize root cause, map blast radius, and generate a concrete fix path. No forms. No dropdowns. Just a conversation that ends with a decision.

## Demo

$ python morpheus.py

Paste your bug report below (press Enter twice when done):

> Users on iOS 16 are getting logged out randomly after ~10 minutes.
> Started happening after last Tuesday's deploy. Stack trace not available.
> Affecting roughly 30% of mobile users based on support tickets.

[Morpheus thinking... analyzing 5 stages]

═══════════════════════════════════════════════════
           MORPHEUS BUG INTELLIGENCE CARD
═══════════════════════════════════════════════════

  SEVERITY       P1 — High  (confidence: 89%)

  ROOT CAUSE     Most likely hypothesis: JWT refresh token
                 silently failing on iOS 16 due to a breaking
                 change in the WebKit storage API introduced
                 in your Tuesday deploy. Auth tokens are
                 expiring without triggering a re-auth flow.
                 Confidence: 82%. Secondary: race condition
                 in session middleware. Confidence: 41%.

  BLAST RADIUS   ~30% of mobile users (iOS 16+). At-risk
                 features: checkout flow, saved preferences,
                 any session-gated content. Risk of churn
                 spike if unresolved past 48 hours.

  OWNER          Recommend: Auth/Identity team. Specifically
                 whoever authored the session middleware
                 changes in Tuesday's deploy.

  REPRO STEPS    1. Use iOS 16 device or simulator
                 2. Log in with a standard user account
                 3. Leave app active (foreground) for 10 min
                 4. Attempt any authenticated action
                 5. Observe forced logout behavior

  DEBUG PATH     Start in: /src/auth/tokenRefresh.js
                 Check: WebKit localStorage behavior change
                 in iOS 16.4+. Review PR diff from Tuesday.
                 Test: Add refresh token expiry logging to
                 isolate whether token is null or rejected.

  SIMILAR BUGS   Matches pattern from Issue #312 (March 2024)
                 — resolved by adding silent refresh fallback.
                 See PR #318 for reference fix.

  OPEN QUESTIONS  - Is this isolated to native WebView or
                   also Safari browser?
                  - Was session middleware touched in Tuesday
                   deploy or only token storage logic?

═══════════════════════════════════════════════════
Output saved to: bug_report_output.md

## How to Use

### Prerequisites
- Python 3.9+
- An API key from Anthropic

### Setup
```bash
git clone https://github.com/abdulmuheeth29/morpheus.git
cd morpheus
pip install -r requirements.txt
```

Create a .env file in the project root:
```bash
ANTHROPIC_API_KEY=your_api_key_here
```

### Run
```bash
python morpheus.py
```

### Example
```
Input:  Users on iOS 16 are getting randomly logged out after ~10 minutes.
        Started after last Tuesday's deploy. Affects ~30% of mobile users.

Output: Bug Intelligence Card with severity P1 (89% confidence), root cause
        hypothesis pointing to JWT refresh failure on iOS 16 WebKit storage
        API, blast radius scoped to session-gated features, recommended owner
        (Auth team), 5-step repro guide, and a targeted debug path.
```

## How It Works

Morpheus runs a 5-stage sequential agentic pipeline, where the output of each Claude API call is passed as context into the next — so the reasoning compounds rather than starting fresh each time. Stage 1 parses and normalizes the raw input into structured signal. Stage 2 detects what context is missing and generates targeted follow-up questions. Stage 3 produces ranked root cause hypotheses with confidence scores and chain-of-thought reasoning. Stage 4 infers blast radius — which users, features, and systems are at risk. Stage 5 assembles the full Bug Intelligence Card: severity rating, owner recommendation, reproduction steps, and a concrete debug path. Each stage is a separate Python function with its own purpose-built system prompt.

## Tradeoffs and Decisions

. Why chained multi-call pipeline over single prompt: A single "analyze this bug" prompt produces flat, overconfident output. Breaking the pipeline into stages forces the model to reason in layers — parse first, hypothesize second, synthesize last. It also makes each stage testable and replaceable independently. The tradeoff is latency (~4–6 seconds vs. ~2 seconds for a single call), which is acceptable for a triage tool where quality matters more than speed.
. Why Claude over GPT-4 for this use case: Claude's extended context window and instruction-following precision make it better suited for multi-stage pipelines where system prompt fidelity across chained calls is critical. The structured output formatting (the Intelligence Card) was also more reliably consistent across runs with Claude Sonnet than with GPT-4o in early testing.
. What I'd do differently: The current similarity-matching (Stage 4) is purely inference-based — it reasons from the bug text rather than actually querying a real issue database. In a production version, I'd integrate a vector store (e.g., Pinecone or Chroma) populated with historical GitHub issues to enable real semantic search across past bugs. That one change would make the blast radius and similar bugs sections dramatically more accurate.

- **Why [choice A] over [choice B]:** [Your reasoning]
- **What I'd do differently:** [Honest reflection]

## What I Learned

. Prompt chaining is an architecture decision, not a prompt trick. How you decompose a complex reasoning task into stages — and what context you pass between them — fundamentally determines output quality. I spent more time on pipeline design than on any individual prompt.
. Confidence scores without calibration are theater. The model will output "87% confidence" because you asked it to, but that number means nothing unless you've validated it against ground truth. For a v1 portfolio project it works as a UX signal, but productionizing this would require a labeled evaluation dataset and calibration work.
. The hardest part wasn't the AI — it was defining "done." A bug report is unbounded input. Deciding when Morpheus had enough context to proceed (vs. asking more questions) required more product thinking than engineering. The 2–3 question limit on follow-ups was a deliberate PM call to prevent the tool from feeling like an interrogation.

## Next Steps

- Integrate GitHub Issues API to pull real open bugs and enable semantic similarity matching against historical issues
- Add a vector store (Chroma or Pinecone) for persistent bug memory and pattern detection across repos
- Build a lightweight web UI (React + FastAPI) so non-technical stakeholders can use it without the terminal
- Add evaluation harness to measure root cause accuracy against a labeled dataset of real resolved bugs
- Support Jira and Linear ticket ingestion as alternative input formats

## Built With

- Anthropic Claude API — claude-sonnet-4-20250514 for all reasoning stages
- Python 3.11
- python-dotenv — environment variable management
- anthropic Python SDK — API client

---
Built by Abdul | [LinkedIn](https://www.linkedin.com/in/abdul-m-mohd/)
