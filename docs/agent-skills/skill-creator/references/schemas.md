# JSON Schemas

Defines the JSON schemas used by skill-creator.

---

## evals.json

Defines the evals for a skill. Located at `evals/evals.json` within the skill workspace.

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's example prompt",
      "expected_output": "Description of expected result",
      "files": ["evals/files/sample1.pdf"],
      "expectations": [
        "The output includes X",
        "The skill used approach Y"
      ]
    }
  ]
}
```

**Fields:**
- `skill_name`: Name matching the skill's frontmatter
- `evals[].id`: Unique integer identifier
- `evals[].prompt`: The task to execute
- `evals[].expected_output`: Human-readable description of success
- `evals[].files`: Optional list of input file paths (relative to skill root)
- `evals[].expectations`: List of verifiable statements (added after initial run)

---

## eval_metadata.json

Metadata for a single eval run. Located at `<workspace>/iteration-<N>/eval-<ID>/eval_metadata.json`.

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": [
    "The output file exists and is non-empty",
    "The correct function was modified"
  ]
}
```

---

## grading.json

Output from the grader. Located at `<run-dir>/grading.json`.

```json
{
  "expectations": [
    {
      "text": "The output includes the name 'John Smith'",
      "passed": true,
      "evidence": "Found in transcript Step 3: 'Extracted names: John Smith'"
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  },
  "claims": [
    {
      "claim": "The form has 12 fillable fields",
      "type": "factual",
      "verified": true,
      "evidence": "Counted 12 fields in output"
    }
  ],
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "The output includes the name 'John Smith'",
        "reason": "A hallucinated document would also pass"
      }
    ],
    "overall": "Assertions check presence but not correctness."
  }
}
```

**Important:** The expectations array must use `text`, `passed`, and `evidence` fields.

---

## comparison.json

Output from the blind comparator. Located at `<run-dir>/comparison.json`.

```json
{
  "winner": "A",
  "reasoning": "Clear explanation of why winner was chosen",
  "rubric": {
    "A": {
      "content": { "correctness": 5, "completeness": 5, "accuracy": 4 },
      "structure": { "organization": 4, "formatting": 5, "usability": 4 },
      "content_score": 4.7,
      "structure_score": 4.3,
      "overall_score": 9.0
    },
    "B": {
      "content": { "correctness": 3, "completeness": 2, "accuracy": 3 },
      "structure": { "organization": 3, "formatting": 2, "usability": 3 },
      "content_score": 2.7,
      "structure_score": 2.7,
      "overall_score": 5.4
    }
  },
  "output_quality": {
    "A": {
      "score": 9,
      "strengths": ["Complete solution"],
      "weaknesses": ["Minor style issue"]
    },
    "B": {
      "score": 5,
      "strengths": ["Readable"],
      "weaknesses": ["Missing fields"]
    }
  }
}
```

---

## analysis.json

Output from the post-hoc analyzer. Located at `<workspace>/analysis.json`.

```json
{
  "comparison_summary": {
    "winner": "A",
    "winner_skill": "path/to/winner",
    "loser_skill": "path/to/loser",
    "comparator_reasoning": "Summary"
  },
  "winner_strengths": ["Clear instructions"],
  "loser_weaknesses": ["Vague guidance"],
  "instruction_following": {
    "winner": { "score": 9, "issues": [] },
    "loser": { "score": 6, "issues": ["Skipped validation"] }
  },
  "improvement_suggestions": [
    {
      "priority": "high",
      "category": "instructions",
      "suggestion": "Be explicit about validation steps",
      "expected_impact": "Would catch errors before output"
    }
  ]
}
```
