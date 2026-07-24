# donate.json

`donate.json` is a machine-readable manifest that an organization publishes at the
well-known URI **`/.well-known/donate.json`** to declare, in a standard form, that it is
a discoverable target for charitable giving — and to carry a small amount of display
metadata about itself. It is the **discovery layer** for agentic charitable giving: a way
for an AI agent or other automated consumer to *find* a donation target and present it
honestly. It is deliberately **not** a payment protocol, an authorization mechanism, or a
source of legal or tax statements.

The full normative specification is in **[`SPEC.md`](SPEC.md)**. The authoritative syntax
contract is **[`schema/donate.schema.json`](schema/donate.schema.json)** (JSON Schema
draft 2020-12), and a standard-library-only Go **reference parser** lives in
**[`parser/`](parser/)**.

## Security-first by design

The manifest is **untrusted, attacker-writable input** — the publisher controls every byte
and may be an adversary impersonating a real charity. The specification is built around
that assumption, and four properties follow from it:

- **Manifests are untrusted data, never instructions or authority.** Free text (names,
  descriptions, extension values) is opaque display data — never executed, never treated
  as an instruction to an agent, never a source of authority.
- **An EIN is a key, not a credential.** A US EIN in a manifest is only a *lookup key* into
  authoritative IRS data (Pub 78 / TEOS / BMF). Its presence proves nothing about
  qualification, good standing, or deductibility; the consumer resolves that out of band.
- **Identity binding is the consumer's determination, never the publisher's claim.** Serving
  a manifest at a domain proves control of that domain, not that the origin is the
  organization it names. A manifest cannot self-assert a "verified" status; binding happens
  only through the consumer's own out-of-band process.
- **No manifest-authored action URLs.** A donation is never initiated by a URL the manifest
  supplies. Any URL in a manifest is informational/navigational only — never an
  authoritative or auto-actionable payment target.

The three things a manifest may **never** be the authority for — **who is paid / recipient
eligibility**, **tax-deductibility**, and **legal / receipt / disclosure copy** — are
resolved exclusively against authoritative sources. See
**[`SPEC.md` §7 (Security Considerations)](SPEC.md#7-security-considerations)**, the heart of
the specification.

## Status

**Version 1.0.** The IANA well-known-URI **provisional registration is in progress** (see
[`SPEC.md` §10](SPEC.md#10-iana-considerations)). The document is written in the style of an
IETF RFC but is not itself an RFC. Within the `1.x` line, changes are additive and
backward-compatible (new OPTIONAL fields only).

## Quickstart

A minimal valid manifest, served at `https://your-org.example/.well-known/donate.json`:

```json
{
  "manifest_version": "1.0",
  "organization": {
    "legal_name": "Example Relief Foundation",
    "country": "US",
    "ein": "12-3456789"
  },
  "donation": { "method": "mediated" }
}
```

`method: "mediated"` means the organization consents to being a discoverable donation
target; the actual gift is initiated by the consuming platform's own controlled flow, keyed
by the *resolved* identity — never by a URL in the manifest. A fuller example is in
[`examples/valid.donate.json`](examples/valid.donate.json).

### Validate against the schema

`schema/donate.schema.json` is a standard JSON Schema (draft 2020-12); validate a manifest
with any conforming validator — for example, with
[`check-jsonschema`](https://github.com/python-jsonschema/check-jsonschema):

```bash
check-jsonschema --schemafile schema/donate.schema.json examples/valid.donate.json
```

### Validate with the reference parser

The Go reference parser in `parser/` is standard-library-only. Run the bundled conformance
suite (the valid and hostile fixtures):

```bash
cd parser
go test ./...
```

To validate your own manifest programmatically, use the two entry points:

```go
import "github.com/givfi/donate-json/parser" // package donate

// Strict: rejects any unknown or authority-asserting field (fails closed).
m, err := donate.Parse(manifestBytes)

// Tolerant: salvages the well-formed core, drops/blanks any overreach, and
// reports everything it dropped for telemetry.
res := donate.Sanitize(manifestBytes)
```

## Conformance modes

A conforming consumer implements at least one of two modes (the reference parser implements
both); see [`SPEC.md` §8](SPEC.md#8-conformance-strict-vs-tolerant):

- **Strict (`Parse`)** — rejects any unknown or authority-asserting field and any constraint
  violation; fails closed. Recommended for ingestion pipelines.
- **Tolerant (`Sanitize`)** — accepts the well-formed core, drops every unknown or
  authority-asserting field, blanks any invalid URL, and reports what it dropped; never fails
  on overreach. Appropriate for best-effort discovery.

In **both** modes the guarantee is identical, and it is a rule on **consumer behavior**
([`SPEC.md` §7.3](SPEC.md#73-the-consumer-behavior-guarantee-normative)): no manifest field
is ever mapped to a payee/eligibility, deductibility, or receipt/legal-copy decision.

## Signing (optional)

Signing is optional and provides **integrity assurance only** — a valid signature proves
authorship by an out-of-band-trusted key and detects tampering; it confers **no authority**
on any field. The format is a detached JWS (RFC 7515), and the signing payload is derived by
**textual excision** (no JSON canonicalization), so signatures cross-verify byte-for-byte
across implementations. See [`SPEC.md` §6](SPEC.md#6-signing-and-integrity).

## Repository layout

| Path | What |
|---|---|
| [`SPEC.md`](SPEC.md) | The v1.0 specification (RFC-style): structure, security model, signing, versioning, IANA registration, legal model, examples. |
| [`schema/donate.schema.json`](schema/donate.schema.json) | JSON Schema (draft 2020-12) — the authoritative syntax contract. |
| [`parser/`](parser/) | Go reference parser (standard-library only): strict `Parse` + tolerant `Sanitize`, signing-payload derivation, and tests. |
| [`examples/`](examples/) | A benign manifest and two hostile fixtures exercised by the tests. |
| [`LICENSE`](LICENSE) | Apache-2.0. |

## Governance

**Change controller: Givmo Charitable Fund ([givmocharitable.org](https://givmocharitable.org)).
Maintained by Givfi, Inc. engineering.**

## Contributing

Issues and pull requests are welcome — bug reports, wording clarifications, additional test
fixtures, and implementation feedback especially. **Normative changes** to the specification
(anything that changes what a conforming producer or consumer must do) go through the change
controller; please open an issue to discuss a normative change before sending a PR.

## License

[Apache-2.0](LICENSE).
