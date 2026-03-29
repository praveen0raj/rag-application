---
description: AI review for requirements analysis
---

Perform AI review of requirements analysis.

## Files to Read Before Starting

Read the following files using the Read tool before starting:
- `.claude/skills/docs/requirements_analysis_review_perspectives.md` — Review perspectives
- `.claude/skills/docs/requirements_analysis_format.md` — Requirements analysis output format
- `.claude/skills/docs/question_format.md` — Question format
- `.claude/skills/docs/domain_knowledge_reference.md` — Domain knowledge progressive reference guide

**Parallel processing principle**: Time-consuming tasks such as code investigation and file reading should be processed in parallel as much as possible.

## Review Perspectives

@.claude/skills/docs/requirements_analysis_review_perspectives.md

## Processing Steps

### 1. Specify Review Target

- Confirm the target EPIC
- Check for the existence of:
    - `epics/{EPIC_NAME}/requirements_analysis.md`

**If local document exists:**
- Proceed to Step 2 without asking, using that file as the review target
- Display "{file path} will be used as the review target"

**If it doesn't exist:**
Use AskUserQuestion to select how to specify the review target:

1. **Specify a URL for review**
    - Copy URL content as-is to local (`epics/{EPIC_NAME}/`) — do not modify or reformat
    - Display "Creating a copy. Will update locally."

2. **Direct input**
    - Create working files (`requirements_analysis.md`, `requirements_analysis_q_and_a.md`)
    - Have user input requirements analysis content

### 2. Review Perspective Check

- At review start, display:
    - "Starting review. If you have specific perspectives you'd like reviewed, send them during execution."
- **If the original input (idea, ticket, conversation, etc.) is available, also check for deviations from the input**
- If the user has specified additional perspectives:
    - Include those perspectives in the review
    - Save additional perspectives to the "Additional Review Perspectives" section in `requirements_analysis_q_and_a.md` (see `.claude/skills/docs/question_format.md` for format)
- Ask about any other unclear points.
- If needed, investigate the codebase.

### 3. Review Results Processing

| Detection | Action |
|-----------|--------|
| Document defects (typos, formatting issues, obvious omissions) | Auto-fix and notify user of corrections |
| Information missing from input | Point out missing information and suggest additions |
| Ambiguous descriptions / unverifiable requirements | Output questions requesting specificity |
| Findings requiring judgment | Output as additional questions in `requirements_analysis_q_and_a.md` |

- See `.claude/skills/docs/question_format.md` for additional question format
- **IMPORTANT: If `requirements_analysis_q_and_a.md` already exists, read it first and APPEND review findings to the end of the file. NEVER overwrite or remove existing questions and answers.** Use the Edit tool to append, not the Write tool which overwrites.
- Prefix review findings with "# Review Findings (Additional Questions)" header to distinguish them from earlier Q&A
- Question numbers must be sequential (continue from existing questions, appended at the end)
- After output, confirm that questions include these options:
    - Other
    - Skip / Consider later
    - Please explain the background of this question
- Display "Q&A file created: {full file path}. Opening file." then run `open {Q&A file path}` using the Bash tool
- Display "Please let me know when you've answered" and use AskUserQuestion with options: "Done", "I'd like investigation results output", "Other"
    - For "I'd like investigation results output": Output results to `requirements_analysis_investigation.md`

### 4. Iterative Review
- When answers are received, process in this order:
    1. Reflect answer content in `requirements_analysis.md` (see `.claude/skills/docs/requirements_analysis_format.md` for format)
    2. Decision record processing:
        - If "Decision Records" section exists, or "don't keep" was previously selected:
            - If exists → Append Q&A content without asking
            - If "don't keep" was selected → Do nothing
        - Otherwise: Ask via AskUserQuestion "Would you like to keep Q&A content as decision records?"
            - "Yes" → Create section and record
            - "No" → Do nothing (don't ask again)
        - When recording, transcribe all options and selected options as-is, don't insert divider lines (---) between questions
    3. Run review again
    4. Repeat from Step 2 until no additional questions remain

### 5. Review Complete

- When no additional questions remain, display:
    - "AI review is complete."
    - "Repeating AI review 2-3 times until no findings remain improves accuracy. Select 'Re-review' to review again."
    - "AI can make mistakes. Domain knowledge and context-dependent aspects cannot be judged by AI, so please verify."
    - "Opening requirements analysis file: {full file path}" then run `open {requirements analysis document path}` using the Bash tool

- **Always** present these options via AskUserQuestion (do not omit):
    1. Re-review
    2. Add perspectives and re-review
    3. Review confirmed
    4. Other
- If review count is 1 or 2, recommend re-review.
  - After that, don't show recommendation level
- If "Re-review" is selected, re-execute from Step 2
- If "Add perspectives and re-review" is selected, ask for perspectives to add then re-execute from Step 2

### 6. Confirm Output Destination

- Have the user choose from:
    1. Update the document at the originally referenced URL
    2. Output to a newly specified URL
    3. Do nothing (local only)
- For Notion, suggest creating it as a sub-page under the specified URL

### 7. Recommend Next Steps

- Use AskUserQuestion to select next action:
    1. Proceed to acceptance criteria (recommended) → Execute `/005-create-acceptance-criteria` via Skill tool
    3. Proceed to technical design → Execute `/007-create-tech-design` via Skill tool
    4. Finish
- This command terminates
