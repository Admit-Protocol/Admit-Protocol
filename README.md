# Admit Protocol

**Status: draft 0.1, November 2026.** One school is live on it. Expect changes; every one is listed in [CHANGELOG.md](CHANGELOG.md).

Admit Protocol is an open standard for applying to a school from a folder on the family's own computer, using the AI assistant the family already has. A school publishes a short instruction file and a schema. The family's agent reads the folder, fills in what the school asks for, and sends nothing until a parent or guardian approves it with an emailed code. The student writes their own essays, and the protocol refuses writing an AI produced. Nothing in it scores or ranks anyone: a person at the school reads every application.

The specification is free to implement. A hosted service that implements it for schools runs at [admitprotocol.com](https://admitprotocol.com); schools pay a flat annual rate for that, families pay nothing, and nothing here requires either.

## What is in this repository

| File | What it is |
|---|---|
| [SPEC.md](SPEC.md) | The specification: what a school serves, what an agent does, and the rules both must keep. |
| [schema/](schema/) | JSON Schema (2020-12) for the application, items, requests and the schema document a school serves. |
| [examples/skill-template.md](examples/skill-template.md) | The instruction file a school serves, with per-school values as placeholders. |
| [examples/vvs-skill-live.md](examples/vvs-skill-live.md) | The same file as one school actually serves it. |
| [CONFORMANCE.md](CONFORMANCE.md) | The checklist a server or an agent can test itself against. |
| [TRADEMARK.md](TRADEMARK.md) | How the name may be used. |
| [LICENSE](LICENSE), [LICENSE-SPEC](LICENSE-SPEC) | MIT for code and schemas; Creative Commons Attribution 4.0 for the specification text. |

## The idea in one screen

```
family's folder  ──agent reads──▶  school's skill.md + /v1/schema
       │                                     │
       │   POST /v1/applications             │  6-digit code ──▶ parent's email
       │   POST .../verify {code}            │  ──▶ application_token
       │   POST .../items  (writing, with attestation)
       │   POST .../items/file (documents)
       │   GET  .../          (checklist: received / missing / next action)
       │   POST .../submit    ──▶ a person at the school reads it
       └── later sessions add more from the same folder
```

Every school serves these under its own base address. The same folder, with a new school's questions answered, applies to the next school.

## Three rules

1. **The student writes.** Agents organize, transcribe and send. They do not draft, rewrite or polish the student's words. Every `writing` item carries an attestation (`written_by: student`, `agent_edited: false`, `confirmed_by: parent|guardian`), and a conforming server refuses writing without it.
2. **People decide.** No scoring, ranking or fit model anywhere in the protocol. A conforming implementation does not add one.
3. **A parent or guardian approves.** The student may run the whole application. The code goes to a parent's email, and nothing is sent until it is entered.

## Implementing it

**As a school or a vendor:** serve a skill file, a schema document and the endpoints in [SPEC.md](SPEC.md) under a base address of your own. Validate against [schema/](schema/). Run through [CONFORMANCE.md](CONFORMANCE.md). Say "implements Admit Protocol" (see [TRADEMARK.md](TRADEMARK.md)) and tell families. If you would rather not build it, the hosted service exists.

**As an agent or agent developer:** nothing to install. Fetch the school's skill file first, then its schema, then follow the flow. The skill file is the contract; when it and this document disagree, the skill file wins for that school.

## Proposing changes

Open an [issue](https://github.com/Admit-Protocol/Admit-Protocol/issues) describing the problem before the change, or start a [discussion](https://github.com/Admit-Protocol/Admit-Protocol/discussions) if it is a question. The bar for a change to the three rules above is very high; the bar for anything else is "a school or an agent needed it." Breaking changes bump `protocol_version` in the schema document.

## Who

Written and first implemented at Verde Valley School, Sedona, Arizona, by its advancement and enrollment office. hello@admitprotocol.com.
