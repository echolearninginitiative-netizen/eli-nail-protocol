# NAIL 1.0

**NAIL: Inter-Agent Message Format** (Native AI Language)
**Version:** 1.0, 8 October 2026. This is the first public version of NAIL. Earlier versions (0.1 to 0.2.1) were used inside ELI and were not published.
**Supersedes:** NAIL 0.2.1 (December 2025), which was used inside ELI and was not published.
**Authors:** v0.2.1 written by Echo and Claude with Glenn Lichtenberger (Project Director). v1.0 revised by the chair (Lumen) after independent reviews by Echo, Nexus and Lumen and a check round. AI-drafted; approved for release by the Project Director on 8 October 2026.
**Licence:** Creative Commons Attribution 4.0 International (CC BY 4.0), https://creativecommons.org/licenses/by/4.0/. You may share and adapt this text, including commercially, if you credit Echo Learning Initiative. It is provided as is, without warranty.
**Status of this text:** experimental. Conformance cannot be claimed against it. A small reference sketch exists (Section 11), but it is a sketch, not a released validator.

---

## 0. What NAIL is, and what it is not

NAIL is an experimental, human-readable format for AI-to-AI messages. A message is a short list of typed blocks that record **declared** sources, confidence and handling restrictions, so that a person or a validator can see where a claim came from, how sure its author was, and how careful the reader must be.

NAIL is not a transport protocol (it can travel inside any channel), not an authentication scheme, and not a safety mechanism by itself. A NAIL message is text. Nothing in it is enforced unless an implementation checks it.

NAIL does **not** guarantee that a claim is true, that reasoning is complete, that an author is who the message says, that provenance is complete, or that prompt injection is resisted. Validation checks only what the message declares. Section 9 lists what NAIL can and cannot detect.

### 0.1 Quick reference

```
KIND:id {
  key=value
} [ meta=value ]

Kinds:   M message   C concept   E evidence   S statement   P plan   Q question
Safety:  safe < sensitive < dangerous        (every block carries one)
Rules:   1 declared dependencies   2 confidence ceiling   3 safety never lower than dependencies
         4 repair only what is unambiguous   5 hold what cannot be resolved
```

---

## 1. Purpose and goals

When several AI systems work on one problem, information passes between them in prose. A "maybe" can become a "yes". A warning can drop out of a summary. A conclusion can lose its trail. NAIL is a small convention for slowing those failures down. Its goals:

- **G1 Visible structure.** Claims, evidence, plans and questions are separate, labelled things.
- **G2 Declared provenance.** A block says which earlier blocks it depends on.
- **G3 Handling labels that do not quietly weaken.** A flag attached to a block is carried to what depends on it.
- **G4 Bounded confidence.** A derived claim is not more confident than the claims it rests on.
- **G5 Cautious repair.** Malformed input is repaired only where the intent is unambiguous; otherwise it is held and a question is raised.

G1 to G5 describe what the format asks of authors and validators. They are not results the format achieves on its own.

---

## 2. Syntax

### 2.1 Encoding and grammar (normative)

A message is UTF-8 text. Lines end with LF or CRLF.

```
ALNUM      = "A".."Z" | "a".."z" | "0".."9"
LOWER      = "a".."z"            DIGIT = "0".."9"            DQUOTE = U+0022
char       = any Unicode character except DQUOTE, backslash, CR and LF
escape     = "\\" ( "\\" | DQUOTE | "n" )           (no other escape is valid)

message    = { ws nl } m-block { sep block } { nl ws }      (blank or whitespace-only lines are allowed before and after)
sep        = nl { ws nl }                           (one or more line ends; blank or whitespace-only lines allowed)
m-block    = "M" ":" MSGID ws "{" nl { field nl } "}" ws "[" ws meta ws "]"
block      = KIND ":" ID ws "{" nl { field nl } "}" ws "[" ws meta ws "]"
KIND       = "C" | "E" | "S" | "P" | "Q"
ID         = idchar { idchar }                      idchar = ALNUM | "_" | "-" | "."
MSGID      = AUTHOR "-" DATE "-" COUNTER            (Section 10.1; DATE is 8 DIGITs, COUNTER is 3 or more DIGITs, ASCII only)
field      = ws KEY "=" value
meta       = item { sp item }                       (items separated by one or more spaces or tabs)
item       = KEY "=" value
KEY        = LOWER { LOWER | DIGIT | "_" }

value      = string | list | reference | number | word      (tried in this order)
string     = DQUOTE { char | escape } DQUOTE
list       = "[" ws [ value { ws "," ws value } ] ws "]"
reference  = ( "M" | KIND ) ":" ID [ "@" MSGID ]
number     = [ "-" ] DIGIT { DIGIT } [ "." DIGIT { DIGIT } ]
word       = idchar { idchar }                     (a token that matches number is a number, not a word)
ws         = { " " | TAB }          sp = ( " " | TAB ) { " " | TAB }          nl = LF | CRLF
```

Rules that go with the grammar:

- A message has exactly one M-block, and it comes first. The M-block's identifier is the message ID (Section 10.1).
- A key may appear only once in a block's payload and only once in its meta, **whatever its value**. A repeated key is invalid, including a repeat with the same value. A field placed in the wrong location (payload versus meta) is invalid; it is not silently overwritten (Section 7).
- The `KIND:ID` pair of a block is unique within its message, so `E:x` and `S:x` are different blocks (a reference always carries its kind). A reference to a block in another message names that message with "@".
- Braces inside strings are text. There are no comments.
- Keys, kinds and enumerated values are case-sensitive. Enumerated values are lower case.
- Unknown keys are preserved and ignored. They never grant anything.
- A block that does not parse is held (Section 7); it is never repaired by guessing.

### 2.2 Field types and required fields

| Key | Where | Type |
|---|---|---|
| purpose, project, name, description, subject, predicate, detail, text | payload | string |
| audience | payload | string, or list of strings |
| steps | payload | list of strings |
| spec_version | payload (M only) | string |
| basis | payload (S) | one of: observation, assertion, proposal |
| about, corrects | payload | list of references (Section 8.2) |
| owner | meta | word or string |
| safety | meta | one of: safe, sensitive, dangerous |
| conf | meta | number from 0 to 1 |
| status | meta | one of: pending, active, blocked, complete |
| prio | meta | one of: low, med, high, urgent |
| derived_from | meta | list of references |
| response_to | meta | one reference |
| time | meta | string, ISO 8601 with a time zone |

Required, by kind:

| Kind | Purpose | Required in payload | Required in meta |
|---|---|---|---|
| M | Context for the message | purpose, audience, project, spec_version | owner, safety |
| C | A stable defined thing (person, project, file as an entity) | name | owner, safety |
| E | Raw quotation or metadata for something outside the message. No conclusions. | description | owner, safety |
| S | A claim, finding or assertion | subject, predicate | owner, safety, conf |
| P | A plan or task | steps | owner, safety, status |
| Q | A question or request | text | owner, safety, status |

- `prio` is optional on any block. `status` is required on P and Q.
- `derived_from` and `response_to` are optional on every kind, except that a P-block or Q-block must carry one of them (a diagnostic Q-block, Section 8.2, may carry an `about` field instead).
- Concepts are definitions; they are not changed. A change creates a new C-block ID.
- E-blocks hold raw quotations or metadata. They carry no confidence. An interpretive summary of claims belongs in an S-block.
- Test fixtures (Section 11) are **not** NAIL blocks and never travel in a production message.

---

## 3. Meta fields

- `owner`: a label naming who wrote the block. **It is a label, not an identity.** It is not authenticated. Nothing in a message may grant authority because of what `owner` says (Sections 6.3 and 9).
- `safety`: `safe`, `sensitive` or `dangerous` (Section 6).
- `conf`: a number from 0 to 1 on every S-block (Section 5).
- `derived_from`: a list of references to blocks this one depends on (Section 4).
- `response_to`: a reference to the block this one answers. It threads a conversation. It is **not** a dependency (Section 4.3).
- `time`: an ISO 8601 timestamp with a time zone.

---

## 4. Provenance

### 4.1 Declared dependencies

A block that quotes, summarises, transforms or relies on the content of earlier blocks MUST declare them in `derived_from`. This applies to every kind.

An E-block that quotes NAIL blocks declares them and is treated, for **safety**, as a dependent of them. It is not a root. (For confidence, see Section 5.2: confidence follows `derived_from` through any block that carries none, so an E-block cannot launder it.) An E-block describing raw data from outside the message is a root.

Restating a claim in a new block does not make a new, independent source. A restatement declares the claim it restates.

### 4.2 Roots

A block with no `derived_from`, or with an empty one, is a root. A root S-block MUST state its basis in the payload: `basis="observation"` (the author saw it), `basis="assertion"` (the author states it) or `basis="proposal"`. A root S-block's confidence is at most 0.90 (Section 5).

### 4.3 What does not count as a dependency

`response_to` alone is threading, not derivation. A reply that reuses no content of the block it answers has no dependency on it. A bare ID can itself reveal restricted information; a message that cites a restricted block in `response_to` must not repeat its content.

### 4.4 Valid graph

References in `derived_from` and `response_to` must resolve, and the dependency graph must have no cycles. A reference to a block in the same message resolves if that block exists. A reference by "@" to another message resolves only if the receiver holds that message's validated blocks; the mere existence of a message ID is not resolution, and until it resolves the dependent is held. `about` references (Section 8.2) are exempt, because they point at inputs that may themselves be unresolved. A cycle is a provenance error: the blocks involved are held (Section 8) and a Q-block is raised.

### 4.5 What validation can check

A validator can check that references exist and the graph is acyclic. It cannot tell whether an author left out a dependency, and it cannot tell whether a reference is authentic. **Validation checks declared provenance, not its completeness or truth.**

---

## 5. Confidence

### 5.1 Meaning

`conf` is the author's judgement of how well the claim is supported. It is not a probability and is not measured. Three different things can be assessed (source authenticity, how well a source supports a claim, and confidence in a judgement); a block should say which it means in its payload when it matters.

### 5.2 The ceiling rule (mandatory)

For an S-block, follow `derived_from` through every block it reaches, **including blocks that carry no `conf`** (such as E-blocks), and collect every S-block reached:

```
conf  <=  min( conf of every reachable S-block )
conf  <=  0.90   for every S-block
```

- If no scored S-block is reachable, only the 0.90 cap applies. 1.0 is never inferred.
- An unscored block does not break the chain. A claim at 0.4 that is summarised through an E-block and then restated at 0.9 is invalid.
- **There is no 1.0 in this version.** Every S-block's `conf` is at most 0.90. (Earlier drafts reserved 1.0 for primary-source checks; that exception is removed until its required fields and its interaction with the ceiling are defined.)
- A machine-generated record (an alert or a hold record, Section 8.2) records its deterministic result in a `result` payload field. If it carries `conf` at all, that is the author's judgement, capped at 0.90, like any other.

### 5.3 Revising confidence

- A revision is a new S-block that names the original in `derived_from` and states its reason. The original is kept.
- A downward revision is always allowed.
- **This version defines no confidence-uplift exception.** A revision keeps its dependencies and obeys 5.2 even when new evidence or human approval is supplied, so it cannot exceed the original. A future version may define a separately validated reassessment rule.

---

## 6. Safety

### 6.1 The three flags

| Flag | Meaning |
|---|---|
| safe | No handling concern |
| sensitive | Handle with care; limit distribution |
| dangerous | Everything `sensitive` requires, **and** a human must review before any action is taken on it |

The order is total: safe < sensitive < dangerous. `dangerous` includes the distribution limits of `sensitive`. Trauma-related material has no separate flag: it is `sensitive` or `dangerous` according to what a human decides.

### 6.2 The lower-bound rule (mandatory)

```
safety of a block  >=  the highest effective safety among its declared dependencies
```

"Effective" means the flag as established through Section 6.3. A block may be marked higher than its dependencies. It may not be marked lower. A block marked lower is a downgrade: it is held (Section 8) and an alert is recorded with the source ID, the flag received and the minimum required.

- A teaching or training reference to a block inherits at least that block's flag. It is not set to a fixed flag.
- A hold record or alert that quotes held content carries the most restrictive flag of that content.
- An author may mark a **new** block higher than its dependencies at any time. An existing block is never changed to raise or lower it; a new block is written instead.

### 6.3 Lowering a flag (reclassification)

A flag on an existing block can be lowered only by a **verified human reclassification**:

- It is an S-block with `predicate="safety_reclassification"` and payload fields `target` (a reference), `old_flag`, `new_flag`, `scope` (the context in which the new flag applies) and `approval_ref` (the identifier the trusted approval mechanism assigned to this decision, which also fixes its order). There is no separate block kind.
- It is valid only if the operator's trusted approval mechanism, outside the message text, has confirmed that a named human with that authority approved **this exact record**: the target, old flag, new flag, scope and `approval_ref` together, not the identifier alone. The text `owner=...` is never enough.
- An AI may propose a reclassification. It cannot authorise one.
- The original block is unchanged. A verified reclassification applies only when the receiver's **evaluation context** (for example `internal` or `public`) matches its `scope`, or its scope is `all`. For a block evaluated in a context it does not match, the original flag stays effective. All verified decisions that match the context are resolved **together** for a target, across scopes, and the verified decision with the latest usable `approval_ref` sets the **effective** flag. If there is no usable order, the most restrictive flag among them applies. The result does not depend on the order of the blocks in the message.
- If two verified reclassifications overlap in scope with no usable order, or the order is missing, the **more restrictive** flag applies until the conflict is resolved.
- The reclassification record itself keeps the target's original handling restrictions: it carries the target's original flag.

### 6.4 Human data and consent

- This rule covers **private or non-public material about an identifiable person**, for example private messages, recordings, case files, or health or location data, and anything gathered about a person beyond their public role. Ordinary discussion of a person's published work or public statements is outside it.
- Before such material is used in public content or teaching content, a consent S-block MUST exist for that material, that purpose and that audience, confirmed through the trusted mechanism of 6.3. Without it, release is blocked. Internal use is not blocked by this rule.
- Consent does not override a safety flag.
- Any statement that attributes motives or actions to a real person must be supported and phrased with the uncertainty it deserves.
- An operator may adopt a wider consent policy. That is a policy of the operator, not a requirement of this format.

### 6.5 Retention and deletion

NAIL operates within the implementer's retention and deletion obligations. A legal or policy obligation to delete personal data overrides any rule here that says content is kept. When content is deleted, the implementation records a tombstone: an S-block with `predicate="deleted"` stating what class of content was deleted, when, and under what authority, without repeating the content.

### 6.6 Migrating from v0.2.1

The old four values map as: safe → safe; sensitive → sensitive; critical → dangerous; trauma → dangerous. A human may later reclassify a particular block from dangerous to sensitive (6.3). The mapping errs towards the more restrictive flag because flags do not go down by themselves.

---

## 7. Repair

When a message is malformed, the receiver repairs only what is unambiguous. It keeps the bytes it received and never mutates them.

- **R1** Repair only when intent is clear.
- **R2** Never guess. If a guess is needed, hold the block and raise a Q-block.
- **R3** Keep the received text, in an `original_` field of the correcting block.
- **R4** Do not change blocks that were not malformed.

**Syntactic normalisation (permitted; confidence is not changed):** the closed alias table below, and moving a meta field that was placed in the payload (or the reverse) to its correct location. Both are recorded in a correcting block.

| Input | Normalised to |
|---|---|
| `status=done` | `status=complete` |
| `status=working` | `status=active` |
| `state=...` | `status=...` |

**Everything else is held, not repaired:** an unknown value for a controlled field (for example `status=maybe`), a repeated key (even with equal values), a wrong-typed value, an unclosed brace, a missing required field. The block is held and a Q-block asks. Any change beyond normalisation requires the author to supply a **corrected, linked block** (payload field `corrects`) with an author-confirmed `conf` where the kind carries one. A receiver never manufactures `conf` for a block of kind E, C, P or Q.

A reference that does not resolve: the dependency stays unresolved after repair, and the hold remains until a resolved reference is supplied. Removing an unresolved reference from active metadata does not clear the hold.

A missing `owner` is recorded as `owner=unknown` only if it is the only fault; a Q-block asks who wrote it.

---

## 8. Hold and quarantine

### 8.1 Holds

A block is **held** when it cannot be trusted as it stands: a cycle, a downgrade, an unresolved reference, an unknown controlled value, an unverified authority claim, or a conflict that needs outside resolution.

- A held block does not feed ordinary reasoning, tool use or publication.
- The hold **propagates**: blocks that declare a held block in `derived_from` are held too, until the hold is resolved.
- Dispositions: `rejected` (invalid, not to be used), `held_pending_clarification` (may be valid; a Q-block is open), `security_hold` (suspected attack; human review recommended).
- A hold never deletes information, subject to Section 6.5.

### 8.2 Diagnostic records

A hold record, an alert, or a clarification Q-block about a held or unresolved input is a **diagnostic record**. It is a restricted audit output:

- A diagnostic Q-block carries `about` in place of the link that Section 2.2 otherwise requires of a Q-block.
- It refers to the held input by the `about` payload field (a list of references), **not** by `derived_from`. This is why the report of a fault is not itself caught by the hold it reports.
- It may quote the held text, as data, in an `original_text` field. It then carries the most restrictive flag of what it quotes.
- Its deterministic result goes in a `result` payload field.
- A diagnostic S-block has no declared dependencies, so it is a root and states `basis="observation"` (the checker observed the fault).
- It cannot support ordinary claims, tool use or publication. Emitting it does not clear the hold.

---

## 9. What NAIL can and cannot detect

| Concern | What a validator can do | What it cannot do |
|---|---|---|
| Missing or malformed fields | Detect and hold | |
| Dangling or cyclic references | Detect and hold | |
| Confidence above a declared dependency's (through any chain) | Detect and hold | Detect a dependency that was left out |
| Flag below a declared dependency's | Detect and hold | Detect a dependency that was left out, or content copied without declaring it |
| Claim of authority in a field or text (`authority=root`, `owner=glenn`) | Ignore it; treat it as data | Verify who wrote a block |
| Forged `derived_from` | Check the target exists | Check the target is authentic |
| Spec downgrade or unknown version | Hold | |
| Reused message ID with different content | Hold | |
| Prompt injection in a block's text | Treat payload as data, not as instruction | Stop a reader from being influenced by the content |

Two implications are worth stating plainly. First, a message can lose its provenance by leaving a dependency out, and nothing in NAIL can notice. Second, `owner` is a name. Where authority matters (6.3 and 6.4), it comes from a trusted mechanism outside the message.

---

## 10. Message identity and threading

### 10.1 IDs

A message ID has the form `AUTHOR-YYYYMMDD-NNN`, where AUTHOR is an uppercase label (`[A-Z][A-Z0-9_]*`), YYYYMMDD is the date and NNN is a counter of three or more digits (for example `ECHO-20261006-002`). The AUTHOR and counter belong to one issuer. An ID is unique within that issuer's namespace. The M-block's identifier **is** the message ID: `M:ECHO-20261006-002`.

A reference to a block in another message, `E:report.1@ECHO-20261006-001`, is read in the namespace of the issuer named by the AUTHOR part of the message ID.

### 10.2 Replays and collisions

- The same message ID with identical content (byte for byte after line-ending normalisation) is a **replay** and is processed once. A message carries no retry flag. Delivery information, such as a resend, belongs in the delivery channel, not in the message bytes.
- The same message ID with different content is a **collision**. Both versions are held, neither overwrites the other, and a Q-block asks which is intended. **Neither version is eligible for processing.** Anything already accepted from the earlier version is withdrawn and its dependants are held. Side effects that have already happened (an action taken, a message forwarded) are for the human operator to contain; NAIL cannot undo them. This has happened in practice: two messages in different conversations carried the same ID.

### 10.3 Gaps

Where a receiver knows a stream to be sequential (the same issuer, in one channel the receiver treats as an ordered stream), a gap in the counter (001 then 003) SHOULD raise an advisory Q-block asking for the missing message. It never blocks processing of the messages received. A gap that may be expected (another session, an abandoned draft) is not raised. Whether a gap is expected cannot be decided from message bytes alone, so the stream context is part of the receiver's configuration.

### 10.4 Digests

This version defines no content digest. A future version may define one, with its location in the message, its algorithm and the exact bytes it excludes. A digest must come from tooling, not from a language model. A digest detects accidental change; it does not show who wrote a message. Do not use ad hoc digests in the meantime.

### 10.5 Threading

Follow-up messages do not change existing blocks. They create new blocks that refer to the old ones by `response_to` (threading) and `derived_from` (dependency).

---

## 11. Conformance

There is no conformance claim at this version, and no published test corpus. A small reference sketch has been written from this text. It implements a **subset** of the syntax and of the deterministic checks, and the example fixtures below are run against it. What it does not implement either fails closed or is listed here as an omission: owner typing, timestamp and version checks, some `about` and `corrects` constraints, consent (V08), evaluation of records from other messages (a dependent on another message is held), and the trusted approval mechanism (supplied to the sketch as an input). The sketch also returns errors and holds, but does not implement the full lifecycle of diagnostic emission, repair and hold resolution. That the rules can be implemented is shown only for the cases the fixtures cover. It does not make any implementation conformant.

What a future validator and corpus need: a parser for the grammar; the deterministic rules (required fields and types, references, cycles, transitive ceiling, lower-bound safety with verified reclassification, holds and their propagation, ID rules); and fixtures kept in a separate fixture format, with deterministic expected results.

### 11.1 Example fixtures

"Deterministic" means a validator can decide it. "Human" needs judgement. "Limit" is a documented limit of the format.

| ID | Case | Expected | Kind |
|---|---|---|---|
| V01 | Safe source, safe derived block | Valid | Deterministic |
| V02 | Sensitive source, derived block marked safe | Downgrade; hold; diagnostic record names source, flag received, flag required | Deterministic |
| V03 | Mixed sources (safe, dangerous), derived block marked sensitive | Downgrade; required minimum is dangerous | Deterministic |
| V04 | Derived block, correctly flagged, saying "no sensitive content" | No parser result. Hold only an explicit release or reclassification claim that nobody has verified; otherwise ask | Human |
| V05 | `derived_from` names a block that does not exist | Hold; Q-block asks for the source | Deterministic |
| V06 | Provenance cycle | Hold both; Q-block | Deterministic |
| V07 | Reclassification with no verified approval (an AI author, or `owner=...` only) | Not effective; original flag stands | Deterministic |
| V07b | Reclassification confirmed by the trusted mechanism (with `approval_ref`) | Effective within its scope; original block unchanged | Deterministic |
| V07c | Two verified reclassifications overlapping in scope with no order | The more restrictive flag applies | Deterministic |
| V08 | Public teaching use of private personal material, no consent block | Release blocked | Deterministic (given the use context) |
| V09 | Same message ID twice, same content; then same ID, different content | Replay processed once; on collision neither version is eligible and earlier acceptance is withdrawn | Deterministic |
| V10 | Counter gap (001 then 003) in a declared sequential stream | Advisory Q-block; messages not blocked | Deterministic (given stream context) |
| V11 | E-block quoting a sensitive block, declaring `derived_from` | The E-block is at least sensitive | Deterministic |
| V11b | Summary of a sensitive block that does not declare it | Not detectable by the validator | Limit |
| V12 | Reply by `response_to` that reuses none of the content | Valid; no inheritance. A reply that repeats the content is V11 | Deterministic |
| V13 | S-block with conf 0.9 derived from blocks at 0.6 and 0.8 | Invalid; hold the received block; a corrected linked block at 0.6 or lower is needed | Deterministic |
| V14 | Root S-block with conf above 0.90, or with no basis | Invalid | Deterministic |
| V15 | Held block with a dependant | Dependant held | Deterministic |
| V16 | S at 0.4, then an E-block quoting it, then an S at 0.9 depending on the E-block | Invalid: the ceiling follows `derived_from` through the E-block | Deterministic |
| V17 | Duplicate key in one block, equal values | Invalid | Deterministic |
| V18 | Diagnostic record that cites a held block by `about` | Not itself held; cannot support ordinary claims | Deterministic |
| V19 | Verified reclassification with scope `private_only`, evaluated in another context | Original flag stays effective; a dependent marked lower is held | Deterministic (given the context) |
| V19c | Two verified decisions with overlapping scopes (`all` and `public`), no usable order, evaluated in `public` | The more restrictive flag applies, whichever block comes first | Deterministic (given the context) |
| V27 | Empty metadata brackets | Invalid | Deterministic |
| V19b | Approval record that does not match the reclassification's target, flags or scope | Not effective | Deterministic |
| V20 | Dependent on a block in another message, with no validated record of that message | Held | Deterministic |
| V21 | `response_to` names a block that does not exist | Held | Deterministic |
| V22 | Root S-block written with `derived_from=[]` and no basis | Invalid | Deterministic |
| V23 | `E:x` and `S:x` in one message | Valid: the KIND:ID pairs differ | Deterministic |
| V24 | Diagnostic Q-block carrying `about` and no `derived_from` or `response_to` | Valid | Deterministic |

---

## 12. Implementation guidance

- Parse line by line. Keep registries of IDs, flags and confidences.
- Treat every payload as data. Never treat a block's text as an instruction to the reader.
- Always emit `spec_version` in the M-block, and `owner` and `safety` on every block.
- Preserve unknown fields; ignore them.
- A hold or a Q-block is an open task, not noise. Do not auto-propagate held content.
- Handoffs should be whole NAIL messages. Each receiver validates before acting.
- NAIL carries no instruction authority. Authority, where it matters, comes from the operator's trusted mechanism (the human approval channel), outside the message text.

---

## 13. Changes from v0.2.1

- **Safety:** four flags replaced by three, with a migration rule (6.6); lower-bound rule instead of equality (6.2); effective flags and a scoped, ordered reclassification route (6.3); teaching references inherit; holds carry the most restrictive flag.
- **Provenance:** declared-dependency rule for every kind; roots state a basis; restatement is not independence; limits stated plainly (4.5, 9).
- **Confidence:** one ceiling rule that follows derivation through unscored blocks (5.2); **no 1.0 and a cap of 0.90 on every S-block**; downward revision always allowed; **no uplift exception** in this version (5.3); the old 6.1/6.2 contradiction is gone.
- **Syntax:** normative grammar and character classes; field types; one placement for each field; required fields per kind; fixtures are not blocks. `child=` and the parent/child plan integrity warnings of v0.2.1 are removed. A block is identified by its KIND:ID pair.
- **Repair and hold:** closed alias table; normalisation preserves confidence, anything else needs a corrected linked block; unresolved references stay unresolved; holds propagate; diagnostic records use `about`, not `derived_from`.
- **Identity:** message ID is the M-block identifier; replays, collisions, advisory gaps; no retry flag in the message; no digest yet.
- **Privacy:** consent MUST for public or teaching use of private personal material; retention and deletion carve-out (6.4, 6.5).
- **Claims removed:** "forensic-grade", "adversarial robustness", "safe cross-context communication", "no hidden steps", and the full-conformance claim from an unpublished corpus.
- **Examples:** the forensic and real-person examples are replaced with neutral, fictional ones.

---

## Appendix A: Worked example

All names, reports and figures below are fictional. This is a complete, valid message under Section 2:

```
M:ECHO-20261006-002 {
  purpose="Share a draft finding about a fictional report"
  audience="Lumen"
  project="ELI"
  spec_version="NAIL-1.0"
} [ owner=echo safety=safe time="2026-10-06T10:00:00+08:00" ]

E:report.1 {
  description="Fictional 'Example Report', section 3, as retrieved"
  retrieved="2026-10-05"
  report_date="2022"
  location="Example Report, section 3, page 12"
} [ owner=echo safety=safe ]

S:finding.1 {
  subject="Example Report"
  predicate="states_figure"
  basis="observation"
  detail="Section 3 states a figure of 40 percent"
} [ owner=echo safety=safe conf=0.8 derived_from=[E:report.1] ]

S:finding.2 {
  subject="Example Report"
  predicate="figure_may_be_out_of_date"
  detail="The report is dated 2022"
} [ owner=lumen safety=safe conf=0.6 derived_from=[E:report.1,S:finding.1] ]

Q:check.1 {
  text="Can someone confirm the report's date?"
  target="Nexus"
} [ owner=lumen safety=safe status=pending prio=med response_to=S:finding.2 ]
```

An invalid variant, for contrast: if `S:finding.2` were written with `conf=0.9`, it would exceed the 0.8 of a block it depends on. A validator holds it and asks for a corrected block at 0.8 or lower (rule 5.2, fixture V13).

## Appendix B: Known limits

1. A dependency left out of `derived_from` cannot be detected.
2. `owner` is a label. Authorship is not authenticated.
3. A reference proves a block exists, not that it is genuine.
4. Confidence numbers are the author's judgement and are not measured.
5. NAIL does not stop prompt injection. It asks readers to treat payloads as data.
6. No released validator or published corpus exists at this version.
7. Which message gaps are expected is decided by the receiver's configuration, not by the message.
8. A reference to another message is only as good as the receiver's validated copy of that message.

---

© 2026 Echo Learning Initiative™ | Echo Learning Initiative Group Pty Ltd | ABN 42 695 263 935 | [Contact & Legal](https://echolearninginitiative.org/eli-contact-and-legal-page/) | [Terms & Conditions](https://echolearninginitiative.org/eli-terms-and-conditions/) | Text licensed under CC BY 4.0
