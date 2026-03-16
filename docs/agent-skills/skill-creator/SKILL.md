---
name: skill-creator
description: Create new skills, modify and improve existing skills, and measure skill performance. Use when the user wants to create a skill from scratch, edit or optimize an existing skill, run evals to test a skill, or iterate on skill quality. Also use when the user says "turn this into a skill" or wants to capture a workflow as a reusable skill.
---

# Skill Creator

A skill for creating new skills and iteratively improving them, adapted for Tasklet agents.

## High-Level Process

1. Decide what the skill should do and roughly how
2. Write a draft of the skill
3. Create test prompts and run them via subagents
4. Evaluate results — qualitatively with the user, quantitatively with assertions
5. Rewrite the skill based on feedback
6. Repeat until satisfied
7. Push the final skill to the repo

Your job is to figure out where the user is in this process and help them progress. Maybe they want to start from scratch, maybe they already have a draft. Be flexible — if the user says "just vibe with me," skip the formal eval loop.

## Communicating with the User

Pay attention to context cues about the user's technical level. Not everyone knows what "assertions" or "JSON schemas" mean. Briefly explain terms if in doubt. Be conversational, not academic.

---

## Creating a Skill

### Capture Intent

Start by understanding what the user wants. The conversation may already contain a workflow to capture (e.g., "turn this into a skill"). If so, extract answers from conversation history first — the steps taken, corrections made, input/output patterns observed. The user fills gaps and confirms.

Key questions:
1. What should this skill enable an agent to do?
2. When should an agent use this skill? (what user phrases/contexts)
3. What's the expected output format?
4. Should we create test cases? Skills with objectively verifiable outputs (file transforms, data extraction, code generation, fixed workflows) benefit from tests. Skills with subjective outputs (writing style, design) often don't.

### Interview and Research

Proactively ask about edge cases, input/output formats, example files, success criteria, and dependencies. Wait to write test prompts until this is ironed out.

Use web search, repo exploration, and any available tools to research context. Come prepared so the user doesn't have to explain everything.

### Write the SKILL.md

Based on the interview, fill in these components:

- **name**: Skill identifier (lowercase, hyphens)
- **description**: When to use it, what it does. Include both purpose AND trigger contexts. Be a little "pushy" — agents tend to under-use skills, so make it clear when the skill applies. Example: instead of "How to edit the app" write "How to edit the Planora monolith HTML. Use this whenever making code changes, fixing bugs, adding features, or modifying any part of the application — even for small CSS tweaks."
- **the rest of the skill**: Instructions, references, examples

### Skill Writing Guide

#### Anatomy of a Skill

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/    - Executable code for deterministic tasks
    ├── references/ - Docs loaded into context as needed
    └── assets/     - Files used in output (templates, icons, fonts)
```

#### Progressive Disclosure

Skills use a three-level loading system:
1. **Metadata** (name + description) — Always readable at a glance (~100 words)
2. **SKILL.md body** — Read when the skill is needed (<500 lines ideal)
3. **Bundled resources** — Read on demand (unlimited size)

Key patterns:
- Keep SKILL.md under 500 lines. If approaching this, add hierarchy and pointers to reference files.
- Reference files clearly from SKILL.md with guidance on when to read them.
- For large reference files (>300 lines), include a table of contents.

**Domain organization**: When a skill supports multiple domains/variants, organize by variant:
```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```
The agent reads only the relevant reference file.

#### Writing Patterns

Use the imperative form in instructions.

**Defining output formats:**
```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**Examples pattern:**
```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

#### Writing Style

Explain the **why** behind instructions rather than stacking rigid MUSTs. Today's LLMs are smart — they have good theory of mind and when given context about reasoning, they go beyond rote instructions. If you find yourself writing ALWAYS or NEVER in all caps, that's a yellow flag. Reframe with reasoning when possible.

Write a draft, then look at it with fresh eyes and improve. Generalize — don't overfit to specific examples.

#### Principle of Lack of Surprise

Skills must not contain anything that would surprise the user in intent if described. No malware, exploits, or deceptive content.

### Test Cases

After writing the skill draft, create 2-3 realistic test prompts — the kind of thing a real user would actually say. Share them with the user for review.

Save test cases to `evals/evals.json` inside the skill workspace:

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's task prompt",
      "expected_output": "Description of expected result",
      "files": []
    }
  ]
}
```

Don't write assertions yet — just the prompts. Draft assertions in the next step while tests run.

See `references/schemas.md` for the full schema.

---

## Running and Evaluating Test Cases

Put results in `<skill-name>-workspace/` as a sibling to the skill directory. Within the workspace, organize by iteration (`iteration-1/`, `iteration-2/`, etc.) and within that, each test case gets a directory (`eval-0/`, `eval-1/`, etc.).

### Step 1: Run Test Cases via Subagents

For each test case, create and run a subagent that:
1. Reads the skill's SKILL.md
2. Follows its instructions to accomplish the test prompt
3. Saves outputs to the workspace directory

Create a subagent markdown file for the executor. The subagent should:
- Read the skill file from the specified path
- Execute the test prompt following the skill's instructions
- Save all outputs to the specified output directory
- Write a brief execution log noting what steps were taken

**With-skill run:** The subagent reads and follows the skill.
**Baseline run** (when creating a new skill): Same prompt, no skill. This shows whether the skill actually adds value.
**Baseline run** (when improving): Use the previous version of the skill as baseline.

Run with-skill and baseline in the same step when possible.

Write an `eval_metadata.json` for each test case:
```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### Step 2: Draft Assertions While Reviewing

While reviewing test outputs, draft quantitative assertions for each test case. Good assertions are objectively verifiable. Subjective skills are better evaluated qualitatively.

Update `eval_metadata.json` and `evals/evals.json` with the assertions.

### Step 3: Grade Results

For each test run, evaluate assertions against outputs. You can do this:
- **Via subagent**: Create a grader subagent following `agents/grader.md` instructions
- **Inline**: For simple cases, grade directly in context
- **Via script**: For programmatically checkable assertions, write and run a script in the sandbox

Save results to `grading.json` in each run directory. The grading.json expectations array must use the fields `text`, `passed`, and `evidence`.

### Step 4: Present Results to User

Present results directly in conversation or via file preview:

For each test case, show:
- **Prompt**: The task that was given
- **Output**: What the skill produced (show files via preview if needed)
- **Grades**: Assertion pass/fail results
- **Previous Output** (iteration 2+): What changed from last time

Ask for feedback: "How does this look? Anything you'd change?"

For larger eval sets, consider building an instant app to display results interactively.

### Step 5: Collect Feedback

The user provides feedback in chat. Empty feedback means they thought it was fine. Focus improvements on test cases where the user had specific complaints.

---

## Improving the Skill

This is the heart of the loop. You've run test cases, the user has reviewed results, and now you improve.

### How to Think About Improvements

1. **Generalize from feedback.** The skill will be used many times across different prompts. Don't overfit to your test examples. Rather than fiddly changes or oppressive MUSTs, try different metaphors or recommend different working patterns.

2. **Keep it lean.** Remove things that aren't pulling their weight. Read the execution logs — if the skill makes the agent waste time on unproductive steps, cut those parts.

3. **Explain the why.** Try to explain the reasoning behind every instruction. Even if user feedback is terse, understand the actual problem and transmit that understanding into the instructions.

4. **Look for repeated work.** If all test runs independently wrote similar helper scripts or took the same multi-step approach, bundle that script in `scripts/` and tell the skill to use it.

### The Iteration Loop

After improving:
1. Apply improvements to the skill
2. Rerun all test cases into a new `iteration-<N+1>/` directory
3. Present results to user (show what changed from previous iteration)
4. Collect feedback, improve again, repeat

Keep going until:
- The user says they're happy
- Feedback is all positive
- You're not making meaningful progress

---

## Advanced: Blind Comparison

For rigorous A/B comparison between skill versions, use a blind comparison via subagent:
1. Give two outputs to a comparator subagent (following `agents/comparator.md`) without revealing which skill produced which
2. The comparator judges quality purely on output merit
3. Then run an analyzer subagent (following `agents/analyzer.md`) to understand why the winner won

This is optional. The human review loop is usually sufficient.

---

## Pushing Skills to the Repo

When the skill is finalized:

1. Organize the skill directory:
   ```
   docs/agent-skills/<skill-name>/
   ├── SKILL.md
   ├── references/  (if needed)
   └── agents/      (if needed)
   ```

2. Push to the repo on the main branch
3. Don't push workspace/eval directories — those are working files only

---

## Reference Files

The `agents/` directory contains instructions for specialized subagents. Read them when spawning the relevant subagent:

- `agents/grader.md` — How to evaluate assertions against outputs
- `agents/comparator.md` — How to do blind A/B comparison between two outputs
- `agents/analyzer.md` — How to analyze why one version beat another

The `references/` directory has additional documentation:
- `references/schemas.md` — JSON structures for evals.json, grading.json, etc.

---

## Core Loop Summary

1. Figure out what the skill is about
2. Draft or edit the skill
3. Run test prompts via subagents
4. Evaluate outputs with the user (qualitative review + quantitative assertions)
5. Repeat until satisfied
6. Push the final skill to the repo

Use your task list to track progress through these stages.
