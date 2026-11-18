# Admit Protocol specification

Version 0.1-draft. November 2026. Text licensed CC BY 4.0; schemas and examples MIT.

The words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

## 1. Parties and terms

- **Family**: the applicant and their parents or guardians. The **student** is the applicant. A **parent** is a parent or legal guardian.
- **Agent**: software acting for the family on the family's own device or account: a coding assistant, a desktop or mobile assistant with connector support, or any client that can read a URL and call an HTTPS API.
- **School**: the institution receiving the application. A **server** is the school's implementation of this specification, run by the school or by a vendor on its behalf.
- **Folder**: the family's documents, kept on the family's device. The folder is the application; the server receives copies of what the family chose to send.
- **Item**: one thing attached to an application: a piece of writing, a document, a recommendation.
- **Base URL**: the address a school serves its endpoints under, for example `https://admitprotocol.com/vvs`. Every path below is relative to it.

## 2. The three rules

These apply to servers and agents alike. An implementation that breaks one does not conform, whatever else it does.

**2.1 The student writes.** An agent MUST NOT draft, rewrite, polish, suggest wording for, or otherwise produce or alter the student's writing, even when asked. An agent MAY ask the student questions, point at files, and transcribe writing the student produced by hand or elsewhere, without changing it. Every `writing` item MUST carry the attestation in section 6, and a server MUST refuse a `writing` item without it.

**2.2 People decide.** A server MUST NOT score, rank, grade, classify by merit, or otherwise evaluate applicants or their writing by software, and MUST NOT expose any such signal to the school's staff or to agents. Validating fields, checking word counts, and detecting that an application is a test or duplicate (section 10) are not evaluation.

**2.3 A parent approves.** A server MUST send the verification code only to the address given as the parent's, MUST refuse to treat an application as verified until that code has been presented, and MUST NOT accept items or a submission before verification. An agent MUST NOT present a parent's code it obtained other than from that parent, and MUST read the full application back to the family before starting it and before submitting it.

## 3. What a school serves

A server MUST serve, under its base URL:

| Method | Path | Purpose |
|---|---|---|
| GET | `/skill.md` | The skill file (section 4). |
| GET | `/v1/schema` | The schema document (section 5). |
| POST | `/v1/applications` | Start an application. Sends the code. |
| POST | `/v1/applications/{id}/verify` | Present the code. Returns the application token. |
| POST | `/v1/applications/{id}/resend` | Send a new code. |
| PATCH | `/v1/applications/{id}` | Change fields other than the verified parent. |
| POST | `/v1/applications/{id}/items` | Add a text item (section 6). |
| POST | `/v1/applications/{id}/items/file` | Add a file (section 7). |
| GET | `/v1/applications/{id}` | Status: the checklist and next action (section 8). |
| POST | `/v1/applications/{id}/submit` | Hand the application to the school. |
| POST | `/v1/applications/{id}/withdraw` | Withdraw it. |

A server SHOULD also serve `/v1/starter` (section 11) and `/v1/applications/{id}/requests` (section 9), and MAY serve an MCP server at `/mcp` exposing the same operations as tools, one to one, with the application token as a tool argument (section 12).

Transport is HTTPS. Request and response bodies are JSON (`application/json`), except the file upload. A server MUST serve `/skill.md` and `/v1/schema` without authentication.

## 4. The skill file

`GET /skill.md` returns Markdown an agent reads before anything else. It MUST begin with a YAML front-matter block with `name`, `description` and `version`, and MUST state, in plain language an agent will follow:

1. the three rules of section 2, including that the AI does not write for the student;
2. the flow of section 13, step by step;
3. the item types the school accepts and which are required to submit;
4. the school's writing prompts, each with its key, word range if any, whether it is required, and which prompts are alternatives;
5. where to keep the application token (section 5.3);
6. the fee, if any, and that the agent never handles payment;
7. what happens after submission, so the agent can tell the family.

The skill file is the contract for that school. Where it and this specification disagree, an agent follows the skill file for that school and reports the disagreement to the family. [examples/skill-template.md](examples/skill-template.md) is a complete example.

## 5. The schema document

`GET /v1/schema` returns a JSON object matching [schema/schema-document.schema.json](schema/schema-document.schema.json). It names the protocol and version, the school, its base URL, skill and MCP addresses, the fee, and carries four JSON Schemas (`application`, `application_patch`, `text_item`, `file_item`), the `item_types`, the `writing_prompts`, a one-line `writing_rule`, and the `checklist`.

### 5.1 The application

`application` MUST accept at least the fields in [schema/application.schema.json](schema/application.schema.json): `parent` (name, email, relationship, optional phone), `student` (first and last name, optional preferred name, date of birth, current school and grade, city, state or region, country, languages at home, `international`), `entering_year`, `entering_grade`, `boarding_or_day`, optional `financial_aid_interest` and `how_you_found_us`, and `agent` (name, optional version). The enumerations of `entering_year`, `entering_grade` and `boarding_or_day` are per school and come from the live schema. A school MAY add fields. A school SHOULD NOT remove these. A server MUST reject unknown fields rather than silently drop them.

### 5.2 Starting and verifying

`POST /v1/applications` with an application body creates a draft and sends a 6-digit code to `parent.email`. The response carries `application_id`, `status: "draft"` and `next_action`. The server MUST NOT return the code. Codes MUST expire within 15 minutes, MUST work once, and the server MUST lock verification after a small number of wrong attempts and say so.

`POST /v1/applications/{id}/verify` with `{ "code": "123456" }` returns `{ "application_token": "...", "status": "open" }`. The token is scoped to one application at one school.

### 5.3 The token

Every request after verification carries `Authorization: Bearer <application_token>`. An agent MUST store the token in the family's folder at `.admit-protocol/<slug>.token`, where `<slug>` is the school's `school_slug`, MUST NOT print it in chat or logs, and MUST NOT send it to any host other than the school's base URL. Tokens SHOULD expire; 90 days is the reference. A server MUST invalidate the token when the application is withdrawn.

## 6. Text items and the attestation

`POST /v1/applications/{id}/items` with a body matching `text_item`: `type` (one of the family-sent item types), optional `prompt` (a key from `writing_prompts`), optional `title`, `text`, and for writing the attestation:

```json
{
  "type": "writing",
  "prompt": "reading",
  "text": "…the student's words, exactly as written…",
  "attestation": { "written_by": "student", "agent_edited": false, "confirmed_by": "parent" }
}
```

A server MUST refuse (400) a `writing` item whose `attestation` is absent or differs from `written_by: "student"`, `agent_edited: false`, and `confirmed_by` one of `"parent"` or `"guardian"`. A server MUST store the attestation with the item and show it to the school's readers. A server MUST NOT alter the text; word counts are reported, never enforced by truncation or rejection.

Prompts: when the school lists `writing_prompts`, each answer is its own `writing` item naming its `prompt`. Prompts sharing a `group` are alternatives; one answer satisfies the group. A server MUST refuse a `prompt` key it did not publish. A server MAY refuse a second answer to the same prompt or MAY replace the first; it MUST say which.

Reference family-sent types: `writing`, `parent_statement`, `transcript`, `report_card`, `test_score`, `english_competency`, `iep_504`, `awards`, `additional`. Types beginning `recommendation_` are never sent by the family (section 9).

## 7. Files

`POST /v1/applications/{id}/items/file` as `multipart/form-data` with a `file` part and a `meta` part matching `file_item` (`type`, optional `title`). A server MUST accept PDF, JPEG, PNG, HEIC and DOCX up to 20 MB, MAY accept more, and MUST say what it refused and why. A server SHOULD strip location and device metadata from images. A server MUST serve stored files only to the school's authenticated staff.

## 8. Status

`GET /v1/applications/{id}` returns the application's `status` (`draft`, `open`, `submitted`, `withdrawn`), its identifying fields, the `checklist`, `missing_required`, `writing` (each prompt with whether it is answered and the word count), and `next_action` in plain words an agent can read aloud.

Each checklist step has a `status` of `missing`, `received`, `reviewed`, `waived` or `school` (a step the school schedules, such as an interview), a `required` flag and an `owner` (`family`, `recommender`, `registrar`, `school`). Step names mirror the school's own system; the reference set is in [schema/reference-checklist.json](schema/reference-checklist.json). An agent MUST read `next_action` to the family and MUST NOT invent requirements the checklist does not state.

Submission: `POST /v1/applications/{id}/submit` succeeds when every step the school marks required is `received` or better. The reference requirement is the application itself and the required writing, nothing more; a school MAY require more and SHOULD require as little as it can. Items MAY still be added after submission. Incomplete applications are normal; a server MUST NOT delete or hide an application because it is incomplete.

## 9. Recommendations and records from third parties

The family does not write, collect or see recommendations. `POST /v1/applications/{id}/requests` with a body matching `recommendation_request` (the recommender's name, role and school email) asks the school to contact that person. The family sees `requested` and `received` on the checklist, never the content. A server that does not yet support requests MUST return 501 with a plain explanation, and the skill file MUST say so.

## 10. Honesty, tests and duplicates

A server MAY detect that an application is a test, a duplicate, or not a real applicant (for example, a parent address at a non-delivering domain, placeholder text, or the same parent fingerprint starting many applications) and MAY mark it for the school's staff and keep it out of the default view. Such marking concerns whether an application is real, never its merit, and MUST be shown to staff as a flag with its reason, never as a score. Staff MUST be able to clear it. This is the only automated judgement this specification permits.

## 11. The starter folder

`GET /v1/starter` returns `{ "folder": "<slug>-application", "files": [{ "path", "contents" }] }`: the files a family's folder should hold for this school, with the question or instruction inside each and the student-writes rule in every writing file. An agent SHOULD offer to write these into an empty folder before asking the family anything else. A server SHOULD also offer the same files as a zip for people.

## 12. MCP

A server MAY expose the operations above as MCP tools over streamable HTTP at `/mcp`, with these names, one per operation: `get_application_schema`, `starter_folder`, `start_application`, `verify_email`, `resend_code`, `update_application`, `add_item`, `add_file`, `get_status`, `submit_application`, `withdraw`, and `request_recommendation` where section 9 is supported. Tools take the application token as an argument named `application_token` until the client ecosystem supports authorization. Tool input schemas MUST be the same schemas as the REST operations.

A hosting platform that serves several schools MAY also expose a **platform server** at its own address (for the reference service, `https://admitprotocol.com/mcp`): `list_schools`, `get_school_instructions` (the school's skill file and schema document), and the tools above with an added `school` argument naming the school's slug. It MUST validate each call against that school's live schema and MUST NOT accept a token issued by one school for another. A school's own server at its base URL remains canonical; the platform server exists so that one connector can reach every school.

## 13. The flow an agent follows

1. `GET /skill.md`, then `GET /v1/schema`. Fields and enumerations come from the schema, not from memory.
2. Read the folder the agent was started in, or the one the family names. If it is empty or nearly so, offer the starter folder (section 11). Map each file to an item type and tell the family what was found. Ask what is missing; do not insist.
3. Build the application and read it back, field by field. Wait for a yes.
4. `POST /v1/applications`. Tell the family a code has gone to the parent.
5. When the parent gives the code, `POST …/verify`. Save the token (section 5.3).
6. Add items. Writing goes as text where possible, each with its attestation, confirmed by the parent before sending. Show the family every file before sending it.
7. `GET …` and read `next_action` aloud.
8. When the required pieces are in and the family says so, `POST …/submit`, and tell the family what happens next, from the skill file.
9. In later sessions, read the token from the folder, `GET …`, and add what is new.

An agent MUST NOT search the web for the student, fill gaps with guesses, send anything the family has not seen, or carry the family's data to any host other than the school's base URL.

## 14. Errors

Error bodies are `{ "error": "plain words", "details": [ … ] }`. 400 for a rejected body, with each rejected field named in `details`; 401 for a missing, wrong or expired token; 403 for a token that is valid but not for this application; 404 for an unknown application; 409 for an action the application's state does not allow; 413 for a file over the limit; 415 for a type not accepted; 429 for too many requests; 501 for an operation the school does not yet support. Messages are written for a person; an agent reads them to the family.

## 15. Privacy and retention

A server MUST serve the school's privacy policy and retention rule at addresses the skill file names. A server MUST store the parent's email address so that it cannot be read from the database alone (the reference stores a keyed fingerprint for lookup and a ciphertext for display to staff). A server MUST let the family withdraw at any time, and withdrawal MUST delete the documents and personal fields at once. A server MUST keep a record of every access to an application by staff and show it to the school. A server MUST NOT use application content to train models, combine it across schools, or disclose it to anyone but the school.

## 16. Versioning

`protocol_version` in the schema document names the version a server implements. Within a major version, servers add fields and item types without removing any; agents ignore fields they do not know. A breaking change bumps the major version, and a server SHOULD keep serving the previous one for a season. The skill file carries the same version.

## 17. Conformance

A server conforms when it serves everything marked MUST in sections 2 to 8 and 13 to 16 under one base URL and passes [CONFORMANCE.md](CONFORMANCE.md). An agent conforms when it follows sections 2, 5.3 and 13. Either may say it "implements Admit Protocol 0.1" ([TRADEMARK.md](TRADEMARK.md)).
