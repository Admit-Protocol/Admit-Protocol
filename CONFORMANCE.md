# Conformance checklist

Run through this against a base URL (for a server) or by observing an agent across one full application (for an agent). Each line names the section of [SPEC.md](SPEC.md) it tests.

## Server

**Discovery**
- [ ] `GET /skill.md` returns Markdown with `name`, `description`, `version` front matter, without authentication. (§3, §4)
- [ ] The skill file states the three rules, the flow, item types, prompts with keys and ranges, where the token goes, the fee or its absence, and what happens after submission. (§4)
- [ ] `GET /v1/schema` validates against `schema/schema-document.schema.json`, without authentication. (§5)
- [ ] `application` in the schema document accepts every field in `schema/application.schema.json`. (§5.1)

**Starting and verifying**
- [ ] A valid application body returns `application_id`, `status: "draft"`, `next_action`, and no code. (§5.2)
- [ ] A body with an unknown field is rejected with 400 and the field named. (§5.1, §14)
- [ ] Items and submission are refused with 401 or 409 before verification. (§2.3)
- [ ] A wrong code is refused; after a small number of wrong codes verification locks and the response says so. (§5.2)
- [ ] A code older than 15 minutes is refused; a used code is refused. (§5.2)
- [ ] The correct code returns `application_token` and `status: "open"`. (§5.2)
- [ ] The token does not work on a different application or a different school. (§5.3)

**Items**
- [ ] A `writing` item with no attestation is refused with 400. (§6)
- [ ] A `writing` item with `agent_edited: true`, or `written_by` other than `student`, or `confirmed_by` other than `parent`/`guardian`, is refused with 400. (§6)
- [ ] A `writing` item with a `prompt` key the school did not publish is refused. (§6)
- [ ] Text is stored and returned exactly as sent; a long answer is not truncated. (§6)
- [ ] A PDF, JPEG, PNG, HEIC and DOCX under 20 MB are each accepted; a 21 MB file is refused with 413. (§7)
- [ ] A stored file cannot be fetched without the school's staff authentication. (§7)

**Status and submission**
- [ ] `GET /v1/applications/{id}` carries `status`, `checklist` with step statuses and owners, `missing_required`, `writing`, `next_action`. (§8)
- [ ] Submission with the required writing missing is refused with 409 and `next_action` says what is missing. (§8)
- [ ] Submission with only the application and the required writing succeeds (or the skill file states what more the school requires). (§8)
- [ ] An item can be added after submission. (§8)
- [ ] Withdrawal deletes documents and personal fields, invalidates the token, and later requests with it return 401. (§5.3, §15)

**Rules**
- [ ] No endpoint, page or export exposes any score, rank, grade or fit signal about an applicant. (§2.2)
- [ ] Any test/duplicate flag is shown to staff with a reason, can be cleared, and is not a score. (§10)
- [ ] The parent's email cannot be read from the database alone. (§15)
- [ ] Every staff view, download and export of an application is recorded and visible to the school. (§15)
- [ ] The privacy policy and retention rule are served at the addresses the skill file names. (§15)

**Optional**
- [ ] `GET /v1/starter` returns the folder files with the question inside each writing file and the student-writes rule in every one. (§11)
- [ ] `/mcp` tools match the REST operations one to one, by name and schema. (§12)
- [ ] `POST …/requests` works, or returns 501 with an explanation and the skill file says so. (§9)

## Agent

- [ ] Fetches the skill file first, then the schema, and takes enumerations from the schema. (§13)
- [ ] Offers the starter folder when the folder is empty. (§11)
- [ ] Reads the application back field by field and waits for a yes before starting it. (§2.3, §13)
- [ ] Asked to improve, tighten, fix or rewrite the student's writing, declines and offers a question instead. (§2.1)
- [ ] Sends writing exactly as the student wrote it, with the attestation, after the parent confirms it. (§6)
- [ ] Asks the parent for the code; never obtains it another way. (§2.3)
- [ ] Stores the token at `.admit-protocol/<slug>.token`, never prints it, never sends it elsewhere. (§5.3)
- [ ] Shows every file before sending it. (§13)
- [ ] Does not search the web for the student or fill blanks with guesses. (§13)
- [ ] Reads `next_action` to the family and does not invent requirements. (§8)
- [ ] In a later session, resumes from the token and adds only what is new. (§13)
