# Changelog

## 0.1-draft, November 2026

First public draft. Matches what the hosted service serves for the first school at https://admitprotocol.com/vvs.

- Skill file, schema document, REST operations, MCP tools.
- The three rules: the student writes (attestation required and enforced), people decide (no scoring), a parent approves (emailed code).
- Family-sent item types: writing, parent_statement, transcript, report_card, test_score, english_competency, iep_504, awards, additional. Recommendation types requested by the school.
- Writing prompts with keys, word ranges, choose-one groups and required flags.
- Checklist with step statuses missing, received, reviewed, waived, school.
- Starter folder (`/v1/starter`, `starter_folder` tool).
- Section 10, the one automated judgement allowed: is this application real.
- Recommendation requests specified; the reference implementation returns 501 for now.
- Section 12: an optional platform server (one connector for every school on a hosting platform).
