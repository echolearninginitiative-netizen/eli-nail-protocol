# NAIL 1.0: Inter-Agent Message Format

**NAIL** (Native AI Language) is a small, human-readable format for AI-to-AI messages. A message is a short list of typed blocks that record **declared** sources, confidence and handling restrictions, so that a person or a validator can see where a claim came from, how sure its author was, and how careful the reader must be.

Version 1.0 is the first public version. It was written by Echo (ChatGPT) and Claude with Glenn Lichtenberger, Project Director of the [Echo Learning Initiative](https://echolearninginitiative.org), and reviewed by other AI systems under human direction. Final decisions rest with a human.

## Read it
- [NAIL_1.0_Inter-Agent_Message_Format.md](NAIL_1.0_Inter-Agent_Message_Format.md) (also as [PDF](NAIL_1.0_Inter-Agent_Message_Format.pdf))

## What it is not
NAIL is experimental. It is not a transport protocol, an authentication scheme or a safety mechanism, and it guarantees nothing by itself. Validation checks only what a message declares. Section 0 and Section 9 of the specification list what it can and cannot detect. There is no released validator and no conformance suite yet.

## Is it new?
We do not claim it is. Several of its ideas have precedent (agent-communication languages, information-flow labels, confidence caps). In a limited search we did not find another small readable format combining them, but we may have missed one. If you know of prior art, please tell us.

## Feedback is very welcome
Please open an [issue](../../issues) for ambiguities, contradictions, prior art, or things that broke when you tried it. We cannot promise to adopt every suggestion or reply quickly, but we will read them.

## Licence
Text: CC BY 4.0. See [LICENSE.md](LICENSE.md).

## Name
Not related to other projects called "NAIL". The subtitle "Inter-Agent Message Format" is to tell them apart.
