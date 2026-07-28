# The `donate.json` Manifest Specification

**Version:** 1.0
**Status:** v1.0. IANA well-known-URI registration is being pursued through the IETF standards process ([§10](#10-iana-considerations)).
**Well-known location:** `/.well-known/donate.json`
**License:** Apache-2.0 (see `LICENSE`)
**Editor:** Givmo Charitable Fund (`givmocharitable.org`) · maintained by Givfi, Inc. engineering

---

## Abstract

`donate.json` is a machine-readable manifest that a website — typically a charity or an organization that raises charitable funds — MAY publish at the well-known URI `/.well-known/donate.json` to declare, in a standard form, that it is a discoverable target for charitable giving and to carry a small set of display metadata about itself.

`donate.json` is the **discovery layer** for agentic charitable giving. An automated agent that encounters a domain can fetch its `donate.json` to learn (a) the organization's self-described identity, including a US EIN used only as a *lookup key*, and (b) that the organization consents to being surfaced as a donation target. It is deliberately **not** a payment protocol, an authorization mechanism, or a source of legal or tax statements.

This document specifies the file location, the JSON structure, a normative security model that treats the manifest as untrusted attacker-writable input, an optional detached-signature integrity mechanism, versioning, and an IANA well-known-URI registration template. A machine-readable JSON Schema (`schema/donate.schema.json`) and a reference parser (`parser/`) accompany this specification.

The specification is open; any site may publish a conforming manifest. Being *listed in, or accepted by, any particular donation platform's catalog* is a separate, curated decision that this specification does not govern (see [§12](#12-relationship-to-consuming-platforms-non-normative)).

---

## Status of This Memo

This document is **version 1.0** of the `donate.json` manifest specification. It is **not** an IETF RFC; it is published independently under the governance described in [§10](#10-iana-considerations). The document is deliberately structured in the style of an RFC because an IETF submission is the intended path: IANA registration of the well-known URI is being pursued through the IETF standards process (an Internet-Draft and a DISPATCH proposal are in preparation — see [§10](#10-iana-considerations) for status). In the interim, `/.well-known/donate.json` operates as an unregistered well-known URI.

---

## 1. Conventions and Terminology

### 1.1 Requirement levels

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

### 1.2 Terminology

- **Manifest** — a JSON document conforming to this specification, served at [§4](#4-well-known-uri-location)'s location.
- **Publisher** — the operator of the origin serving the manifest. The publisher is **untrusted**; see [§7](#7-security-considerations).
- **Consumer** — software that fetches and processes a manifest. A **conforming consumer** implements the normative rules herein.
- **Agentic consumer** — a consumer operated autonomously by an AI agent (e.g. an LLM tool call), as opposed to a human browsing.
- **Consuming platform** — the party that mediates an actual gift on behalf of a consumer. For the reference deployment this is **Givmo**, operated by **Givfi, Inc.**, whose charitable entity is **Givmo Charitable Fund (GCF)**, a 501(c)(3) donor-advised-fund sponsor.
- **Authoritative source** — a data source the publisher does **not** control, used to resolve facts the manifest is not permitted to assert: for US recipient eligibility, IRS Publication 78 / Tax-Exempt Organization Search (TEOS) / the Business Master File (BMF); for legal, receipt, and disclosure copy, the consuming platform's own server-authored, versioned registry.
- **EIN** — US Employer Identification Number. In this specification an EIN is exclusively a *lookup key* into authoritative IRS data. It is never proof of anything.

---

## 2. Design goals and non-goals

**Goals.**

1. Let an agent *discover* that a domain is a charitable-giving target, and obtain enough display metadata to present it honestly.
2. Bind that discovery to a stable identity key (the EIN) that a consumer can resolve against authoritative data **out of band**.
3. Be trivially publishable (a static file) and trivially parseable (plain JSON, one small schema).
4. Fail safe: a hostile or malformed manifest can never escalate into money movement, a false tax statement, or forged legal copy.

**Non-goals.**

1. **Not** a payment or checkout protocol. The manifest carries no authoritative action URL, no payment endpoint, no amount, and no credential (see [§5.4](#54-donation), [§7](#7-security-considerations)).
2. **Not** an authorization mechanism. Nothing in the manifest grants any capability to any party.
3. **Not** a source of eligibility, deductibility, receipt, or disclosure statements. Those are resolved against authoritative sources, never read from the manifest ([§7.2](#72-the-three-forbidden-authorities)).
4. **Not** a catalog-membership grant. Publishing a manifest does not entitle the publisher to appear in any consuming platform's catalog ([§12](#12-relationship-to-consuming-platforms-non-normative)).

---

## 3. The trust model in one paragraph (normative)

**The manifest is untrusted, attacker-writable input.** The publisher controls every byte and may be an adversary impersonating a legitimate charity. A conforming consumer therefore treats the manifest as *data for the discovery and display layer only*, and MUST resolve the following against authoritative sources rather than the manifest: **(a) who may be paid and whether a recipient is eligible; (b) whether any contribution is tax-deductible and to what extent; and (c) the content of any legal disclosure, acknowledgment, or tax receipt.** These are the **three forbidden authorities** ([§7.2](#72-the-three-forbidden-authorities)). Every other normative rule in this document follows from this paragraph.

---

## 4. Well-known URI location

A manifest, if published, MUST be served at:

```
https://{authority}/.well-known/donate.json
```

- The scheme MUST be `https`. Consumers MUST NOT fetch a manifest over cleartext `http`.
- The path is fixed per RFC 8615 (well-known URIs) as `/.well-known/donate.json`.
- The response SHOULD be served with `Content-Type: application/json`. A conforming consumer SHOULD parse a `2xx` response body as JSON regardless of a mislabeled content type, but MUST NOT execute or interpret it as anything other than data.
- A consumer SHOULD follow at most a small, fixed number of HTTP redirects (RECOMMENDED: ≤ 3), MUST require `https` at every hop, and SHOULD treat a cross-origin redirect as a signal to re-derive identity from the final origin.
- Discovery of a manifest at a domain does **not** by itself associate the manifest with any particular organization identity, and MUST NOT be treated as doing so. Identity binding happens only through the consumer's own out-of-band process ([§5.3a](#53a-identity-binding-normative)); an unbound manifest is display-only with no donation affordance.

Fetching guidance (SHOULD): send a `User-Agent` identifying the consumer; set a conservative timeout; cap the response body (RECOMMENDED: ≤ 64 KiB) and reject larger bodies; do not follow non-`https` links found inside the manifest ([§7.6](#76-no-manifest-authored-action-urls)).

---

## 5. Manifest structure

A manifest is a single JSON object. The authoritative contract is `schema/donate.schema.json` (JSON Schema draft 2020-12). This section is the human-readable field reference. Where this prose and the schema disagree, **the schema governs syntax and the Security Considerations govern authority.**

Objects use `additionalProperties: false` in strict validation: unknown fields cause rejection in strict mode and are dropped-and-reported in tolerant mode ([§8](#8-conformance-strict-vs-tolerant)).

### 5.1 Top level

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `manifest_version` | string (SemVer `1.x`) | MUST | Spec version this document targets. A consumer MUST reject a **major** version it does not implement ([§9](#9-versioning)). |
| `generated_at` | string (RFC 3339) | MAY | When generated. Advisory freshness only; never an authority input. |
| `expires_at` | string (RFC 3339) | MAY | Refetch-after hint. Advisory only. |
| `organization` | object | MUST | Self-asserted identity ([§5.2](#52-organization)). |
| `donation` | object | MAY | Discovery mode ([§5.4](#54-donation)). |
| `display` | object | MAY | Cosmetic hints ([§5.5](#55-display)). |
| `contact` | object | MAY | Non-authoritative contact info ([§5.6](#56-contact)). |
| `signature` | object | MAY | Detached-JWS integrity envelope ([§6](#6-signing-and-integrity)). |
| `x_extensions` | object | MAY | Namespaced forward-compat bag; always untrusted ([§5.7](#57-x_extensions)). |

### 5.2 `organization`

Every member is a **claim** by the publisher.

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `legal_name` | string | MUST | Self-asserted legal name. **Display candidate only** — the authoritative name is the resolved IRS record's. |
| `also_known_as` | string[] | MAY | Display aliases / DBA. |
| `country` | string (ISO 3166-1 alpha-2) | MUST | Registration country. v1 authoritative resolution is **US-only**; a non-US value is accepted for display but yields no verified US-tax posture. |
| `ein` | string (`NN-NNNNNNN`) | MAY | US EIN as a **lookup key** into IRS authoritative data. **Presence is not proof** of qualification, good standing, or deductibility. Meaningful only when `country` = `US`. |
| `website` | string (`https://…`) | MAY | Canonical homepage. Navigational only; MUST NOT be used as a donation action target ([§7.6](#76-no-manifest-authored-action-urls)). |
| `logo` | string (`https://…`) | MAY | Logo image URL, for display; fetched (if at all) as an image, never interpreted as a document. |
| `description` | string (≤ 2000) | MAY | Plain-text self-described mission. **Untrusted free text** — treated as attacker-writable data, markup-stripped, provenance-labeled on display, never as instructions. MUST NOT contain deductibility/fee/receipt/legal statements; any such content is ignored ([§7](#7-security-considerations)). |

### 5.3 The EIN is a key, not a credential (normative)

A consumer MUST NOT treat the presence, format-validity, or any manifest-supplied annotation of `ein` as establishing that the organization is a qualified charity, is in good standing, or that gifts are deductible. The consumer MUST resolve the EIN against authoritative IRS data (Pub 78 / TEOS deductibility code; BMF foundation code; Auto-Revocation List) and MUST re-confirm eligibility at the time money would move, not merely at discovery. A manifest whose `ein` fails authoritative resolution is a **research/display-only** record and MUST NOT be surfaced with a donation affordance.

### 5.3a Identity binding (normative)

Discovering a manifest at a domain **MUST NOT** auto-associate that manifest — or its self-asserted `legal_name`/`ein` — with any organization identity. Serving `donate.json` at a domain proves control of that domain, not that the origin is the organization it names ([§7.8](#78-identity-spoofing-and-lookalikes)). Association of a manifest with an organization identity happens **only** through the **consumer's own out-of-band process**. For Givmo specifically, that process is: curation + authoritative IRS resolution of the EIN + (where used) a signing key bound to the organization **at curation time**, verified per [§6](#6-signing-and-integrity).

Until a manifest is identity-bound by such an out-of-band process, a consumer **MUST** treat it as **display-only / research-only, with no donation affordance** — the same posture as an `ein` that fails authoritative resolution ([§5.3](#53-the-ein-is-a-key-not-a-credential-normative)). Concretely, absent **either** an out-of-band-bound signing key **or** an out-of-band domain-control proof accepted by the consumer, the manifest carries no donation affordance.

A manifest **MUST NOT** self-assert its own identity-binding status: there is no manifest-writable "verified", "identity_confirmed", "catalog_accepted", or equivalent field, and a consumer MUST ignore any such field ([§7.5](#75-authority-name-scan-best-effort-telemetry-non-normative)). Binding is always the consumer's determination, never the publisher's claim.

> **Reserved extension point (future revision).** v1.0 deliberately does **not** define an interoperable binding scheme; the consumer-out-of-band requirement above is the complete v1.0 mechanism. An OPTIONAL, interoperable **domain-control-proof + public-key-binding** scheme is a planned additive extension to this section (direction: a domain-control proof — e.g. a DNS record or a second well-known path — plus a standard public-key binding); it will be integrated in a future revision of this specification without altering v1.0 conformance. Consumers implementing v1.0 lose nothing by shipping now: identity binding remains the consumer's out-of-band determination until that extension lands.

### 5.4 `donation`

Declares the **discovery mode**. It deliberately cannot supply an authoritative action URL.

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `method` | enum: `mediated` \| `self_hosted` \| `none` | MUST (if `donation` present) | See below. |
| `self_hosted_url` | string (`https://…`) | MAY (only if `method`=`self_hosted`) | The org's own donation page, **for a human to visit out of band**. |

- **`mediated`** — the organization consents to being a discoverable donation target. The actual gift is initiated by the **consuming platform's controlled flow**, keyed by the resolved identity — never by a URL in this manifest. This is the value that carries donation semantics for an agentic consumer.
- **`self_hosted`** — informational only. The org runs its own donation page. A conforming **agentic consumer MUST NOT** auto-navigate to, transact against, attach any credential to, or pass to a model as a tool target the `self_hosted_url`; it MUST hand any such link to a human out of band ([§7.6](#76-no-manifest-authored-action-urls)).
- **`none`** — identity published, but no opt-in to programmatic donation discovery.

For the reference consuming platform (Givmo), a `mediated` gift is an **irrevocable gift to Givmo Charitable Fund (GCF)**, which grants onward at its discretion; it is **not** a direct gift to the named organization ([§11](#11-legal-model-normative-constraints-on-consumers)). A manifest MUST NOT, and cannot via this schema, assert otherwise.

### 5.5 `display`

Cosmetic, non-authoritative hints: `primary_color` (`#RRGGBB`), `categories` (self-described tags, never used for eligibility/routing), `languages` (BCP-47 tags). All untrusted; subject to the consumer's own sanitization.

### 5.6 `contact`

Optional, **non-authoritative** contact info for display / operator outreach only:

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `support_email` | string (email) | MAY | Self-asserted support address. Display/outreach only; never an authority input. A consumer SHOULD validate its shape and MUST NOT auto-send credentials or PII to it. |
| `support_url` | string (`https://…`) | MAY | Self-asserted support page. Navigational only, subject to the URL rules of [§7.6](#76-no-manifest-authored-action-urls) and the SSRF guard of [§7.9](#79-fetch-time-hardening). |

Both are untrusted claims; neither carries authority.

### 5.7 `x_extensions`

A namespaced object for forward-compatible extensions. Consumers MUST ignore keys they do not understand. **Nothing in `x_extensions` may carry authority** over any of the three forbidden authorities; a conforming consumer treats the entire bag as untrusted display-layer data and MUST NOT map any key or value to an eligibility, deductibility, routing, or legal-copy decision ([§7.3](#73-the-consumer-behavior-guarantee-normative)). The best-effort authority-name scan ([§7.5](#75-authority-name-scan-best-effort-telemetry-non-normative)) surfaces hostile keys for telemetry but is not the guarantee. Keys SHOULD be namespaced (e.g. `vendor:feature`).

---

## 6. Signing and integrity

Signing is **OPTIONAL** and provides **origin-integrity assurance only**. A valid signature proves the manifest was authored by the holder of a key the consumer **already trusts out of band**. **A signature confers no authority on any field** — a signed manifest is exactly as untrusted, with respect to the three forbidden authorities, as an unsigned one. Signing exists so a consuming platform that has previously bound a key to an organization can detect tampering and content drift, not to let a manifest self-certify.

### 6.1 Format

The `signature` object is a **detached JWS** (RFC 7515, compact serialization):

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `alg` | enum: `EdDSA` \| `ES256` \| `ES384` \| `RS256` | MUST | JWS algorithm. Consumers **MUST reject `none`** and MUST reject any symmetric (HMAC) algorithm for third-party manifests. |
| `kid` | string | MAY | Key identifier to select among out-of-band-trusted keys. |
| `jws` | string (`header..signature`) | MUST | Detached JWS compact serialization: `BASE64URL(protected_header)` `..` `BASE64URL(signature)`; the payload segment is empty. |

### 6.2 Signing payload — textual excision, no canonicalization

To keep signatures reproducible **without any JSON-canonicalization dependency** (which is a notorious cross-implementation interoperability hazard — number-formatting and key-ordering divergences make canonicalizers disagree, so signatures fail to cross-verify), the signing payload is defined as a **precise textual excision** of the published bytes, not a re-serialization. There is no re-encoding step and no canonical form to agree on: the signer and the verifier operate on the exact same on-the-wire bytes.

The rule is:

1. A manifest is a UTF-8-encoded JSON object. Call its exact published byte sequence **M**.
2. When a manifest is signed, the `signature` member **MUST be the last member of the top-level object** — i.e. `signature` is the final key before the closing `}` of the root object. (This is the one formatting constraint signing imposes; unsigned manifests are unconstrained.)
3. The **signing payload P** is **M with the entire `signature` member textually removed**, defined exactly as: starting from **M**, delete the byte range that begins at the comma separating the `signature` member from the preceding member and ends immediately before the root object's closing `}`. The result **P** is itself a well-formed JSON object (the same manifest without `signature`).
4. **In both the signed and the unsigned form, P ends at the root object's closing `}`.** Bytes after that closing `}` — trailing whitespace such as a final newline, which is all JSON permits there — are **excluded** from the signing payload. For an unsigned document (the producer's sign-first step, where P is the whole document) this means P runs up to and including the root `}`, not to the raw end of the file; the signed form's excision in rule 3 already ends at the root `}`. This symmetry is what keeps sign-then-append verifiable: a publisher that signs a newline-terminated file and then appends the `signature` member produces a manifest whose excised payload is byte-identical to the payload it signed.
5. **P** is the payload that is signed and verified. The JWS is **detached** (RFC 7515 §A.5): the JWS payload segment is empty on the wire; the signing input is `ASCII(BASE64URL(protected_header))` `.` `BASE64URL(P)`.

Because **P** is obtained by deleting a byte range from **M** — never by parsing and re-serializing — the verifier reconstructs the identical payload the signer used with byte-for-byte fidelity, regardless of language, JSON library, key ordering of the *other* members, number formatting, or whitespace. The publisher signs first (over the object without `signature`), then appends the `signature` member as the last key; the verifier excises it back out.

A verifier MUST reject a signed manifest in which `signature` is not the last top-level member (it cannot reconstruct **P** deterministically otherwise), and MUST reject one whose excised **P** does not parse as a JSON object. The reference parser implements this excision as `SigningPayload`.

*(Rationale for not using RFC 8785 JCS: JCS would also work, but only if made a hard MUST and implemented exactly to spec on every side; the number-canonicalization rules in particular are easy to get subtly wrong. Textual excision has no such surface — it is the simpler, more interoperable option and is what this spec pins.)*

### 6.3 CLI verbs (`givmo manifest validate|sign`)

The `givmo` CLI (a separate deliverable) provides:

- **`givmo manifest validate <file|url>`** — fetches (if a URL) over `https`, parses in strict mode against the schema, runs the best-effort authority-name scan ([§7.5](#75-authority-name-scan-best-effort-telemetry-non-normative)), and, if a `signature` is present and a trusted key is configured, verifies it. Exit code is non-zero on rejection; `--json` emits the structured findings (including every dropped authority claim) for telemetry.
- **`givmo manifest sign <file> --key <key>`** — computes the signing payload ([§6.2](#62-signing-payload--textual-excision-no-canonicalization)) over the manifest **without** a `signature` member, produces a detached JWS with an allowlisted asymmetric `alg`, and appends `signature` as the **last** top-level member. It MUST refuse `alg: none` and symmetric algorithms.

The reference parser's `VerifySignature` performs structural checks (detached form, allowlisted asymmetric `alg`, `none` rejected) and delegates cryptographic verification to a caller-supplied `Verifier`, so the reference core carries no third-party crypto dependency; the CLI wires in a concrete `crypto/ed25519` / `crypto/ecdsa` verifier over its trusted keys.

---

## 7. Security Considerations

> This section is the heart of the specification. Implementers MUST read it in full.

### 7.1 The manifest is untrusted, attacker-writable input

Every field originates from the publisher, who is **not trusted** and may be an adversary — a phishing site impersonating a real charity, a compromised legitimate site, or a genuine charity whose CMS was injected. Consumers MUST assume any string may be crafted to mislead a human reader, to inject instructions into an LLM ("prompt injection"), or to redirect value. A manifest is **data**, never instructions and never authority.

The threats this section addresses — manifest injection, sock-puppet charities, receipt fabrication, and cross-domain agent identity — are the specific attacks a consumer of untrusted manifests must withstand.

### 7.2 The three forbidden authorities

A conforming consumer MUST resolve each of the following against **authoritative sources**, never from the manifest:

1. **Who is paid / recipient eligibility.** Resolved against IRS authoritative data (Pub 78 / TEOS deductibility code as the eligibility floor; BMF foundation code as the grant-classification overlay; Auto-Revocation List as the negative check), never the manifest's `ein` annotations or any self-attestation, and re-confirmed at settlement. Trusting the manifest here can steer value to a non-qualified or fraudulent recipient — a taxable-distribution and control risk for a DAF sponsor ([§11](#11-legal-model-normative-constraints-on-consumers)).
2. **Tax-deductibility.** The manifest MUST NOT state, and a consumer MUST NOT infer from the manifest, that any gift is deductible, to what extent, or to whom. Deductibility is a function of the consuming platform's structure and the donor's circumstances, communicated only through the platform's authoritative copy.
3. **Legal disclosure, acknowledgment, and receipt copy.** All such copy is server-authored by the consuming platform from a versioned, immutable registry, bound per transaction. **No manifest field, and no agent, may supply or influence a receipt, acknowledgment, disclosure, disclaimer, or fee statement.**

The manifest feeds the **discovery/display layer only**. This is a rule on the **consumer's behavior** ([§7.3](#73-the-consumer-behavior-guarantee-normative)), and one way to implement it robustly is a data model in which the three authorities are structurally unrepresentable ([§7.4](#74-implementation-technique-unrepresentable-authority-non-normative)).

### 7.3 The consumer-behavior guarantee (NORMATIVE)

The security guarantee of this specification lives in **consumer behavior**, not in any particular parser's data types. Any conforming consumer — including a third-party consumer written from `schema/donate.schema.json` alone, which admits an open `x_extensions` object and does not itself define an authority-name scan — MUST enforce the following:

- A consumer **MUST NOT** map any manifest field (defined field, unknown field, `x_extensions` content, or any free-text value) to a decision about **who is paid, whether a recipient is eligible, whether any contribution is deductible, or the content of any legal disclosure, acknowledgment, receipt, or fee statement**. Those decisions are resolved exclusively against authoritative sources ([§7.2](#72-the-three-forbidden-authorities)).
- A consumer **MUST** treat `x_extensions`, `organization.description`, and every other free-text or open value as **opaque, untrusted display-layer data**: never as instructions, never as authority, never as routing or eligibility input.
- A consumer **MUST NOT** treat any URL in the manifest as an authoritative action target ([§7.6](#76-no-manifest-authored-action-urls)).
- A consumer **SHOULD** run a best-effort authority-name scan over unknown/extension keys and emit telemetry on matches ([§7.5](#75-authority-name-scan-best-effort-telemetry-non-normative)) — but no security property depends on that scan's completeness; the guarantee is the MUSTs above.

Because these are behavioral MUSTs, they bind every conforming consumer regardless of implementation language, of whether it uses the reference parser, or of whether it derives its types from the schema. The schema deliberately does **not** try to encode "authority" — it cannot, since `x_extensions` is open by design — so the authority constraint is specified here as a consumer obligation.

### 7.4 Implementation technique: unrepresentable authority (NON-NORMATIVE)

One robust way to satisfy [§7.3](#73-the-consumer-behavior-guarantee-normative) — the technique the reference parser uses — is to build a data model in which the three authorities are **unrepresentable**. The reference parser's `Manifest` type contains no member for deductibility, qualification/good-standing, receipt text, or legal copy, so a hostile manifest carrying such members is either **rejected** (strict `Parse`, via `DisallowUnknownFields`) or has those members **dropped and reported** (tolerant `Sanitize`); the data simply has no field to land in. This is a strong and recommended pattern, but it is an *implementation technique*, not the specification's guarantee: a consumer that (say) deserializes into an open map still MUST obey [§7.3](#73-the-consumer-behavior-guarantee-normative)'s behavioral rules. Note this technique covers the **three authorities** specifically (payee/eligibility, deductibility, receipts/legal-copy), which have no schema field at all; it does **not** apply to URLs, which *are* representable fields (`website`, `logo`, `self_hosted_url`, `support_url`) and are non-authoritative by the consumer rule of [§7.6](#76-no-manifest-authored-action-urls), not by absence.

### 7.5 Authority-name scan (best-effort telemetry, NON-NORMATIVE)

As a hostility signal, a consumer SHOULD scan unknown/extension keys for names that *assert* one of the three forbidden authorities (e.g. `deduct*`, `receipt`, `*_disclosure`, `qualified`, `good_standing`, `action_url`, `payto`, `authoriz*`, `token`, `execute`, `skip_verification`, `confirmed_recipient`, `catalog_accepted`, `*override*`) and emit telemetry on each match; the reference parser does this and records each as a `RejectedClaim`. **This scan is deliberately incomplete and no security property depends on its completeness.** It is key-**name**-only and best-effort: it does **not** inspect values, and it will miss authority-named keys it does not enumerate (`payee`, `beneficiary`, `iban`, `routing_number`, homoglyph/unicode-confusable variants, novel coinages, values that assert authority under an innocuous key, etc.). That is acceptable **precisely because** the real guarantee is [§7.3](#73-the-consumer-behavior-guarantee-normative): any such key lands in opaque `x_extensions` (or is an unknown field) and is therefore never mapped to an authority decision, whether or not the scan named it. The scan exists to surface hostile intent for logging/blocking, not to be the defense. Do not treat a clean scan as evidence a manifest is benign.

### 7.6 No manifest-authored action URLs

No URL in the manifest is an authoritative action target. Specifically:

- A donation is **never** initiated by navigating to a URL the manifest supplies. `method: mediated` routes through the consuming platform's own controlled flow, keyed by resolved identity.
- `self_hosted_url` and `website` are **informational**; an agentic consumer MUST NOT auto-navigate to them, MUST NOT pass them to a model as a tool/action target, and MUST NOT attach any credential, token, or PII to any request derived from them. Such links are handed to a human out of band.
- Consumers MUST NOT construct an authenticated request to any manifest-supplied URL. Checkout/session URLs used by the consuming platform are minted **by the platform**, are single-use and secretless, and never originate from the manifest.

This closes the "model-authored URL carrying authority" class: an LLM reading a poisoned manifest cannot be steered to a payment or exfiltration endpoint, because manifest URLs are non-actionable by rule and the platform's real action URLs are not in the manifest.

### 7.7 Prompt-injection and untrusted-text handling

Free-text fields (`organization.description`, `also_known_as`, `display.categories`, any `x_extensions` value) are the injection surface. Consumers, especially agentic ones, MUST:

- Treat these strings strictly as **data**. When such text is placed into an LLM context, it MUST be clearly delimited/encoded as untrusted content, provenance-labeled ("self-described by the organization's website"), and MUST NOT be concatenated into instruction positions.
- **Strip markup** and control characters before display; never render publisher HTML/script.
- Apply the consuming platform's injection screen; that screen SHOULD fail closed (drop the field) rather than pass questionable content through.
- Enforce the length bounds in the schema; reject/truncate over-long fields.

### 7.8 Identity spoofing and lookalikes

A manifest's `legal_name`/`logo`/`website` can impersonate a well-known charity. The `legal_name` is a **display candidate only**; the consumer's displayed identity MUST be the authoritatively-resolved one, and any mismatch between the manifest's claimed name and the resolved IRS record SHOULD be surfaced or SHOULD cause the manifest to be treated as display-only. Domain possession is not identity: serving a manifest at `charity-example.org` does not establish that the origin is the organization behind a given EIN ([§5.3a](#53a-identity-binding-normative)).

### 7.9 Fetch-time hardening

- Consumers **MUST** require `https` for the manifest fetch and every redirect hop, and **MUST reject** any non-`https` hop.
- Consumers **MUST** enforce a response-body size cap **before parsing** (RECOMMENDED: ≤ 64 KiB) — the cap is checked against `Content-Length` and again while streaming the body — and reject a larger body without parsing it. A malformed or oversized body MUST fail safe (no partial authority extraction).
- Consumers SHOULD cap redirects (RECOMMENDED: ≤ 3) and SHOULD set conservative connect/read timeouts.
- Consumers SHOULD guard against decompression bombs (cap the decompressed size, not just the wire size).
- **SSRF (MUST for agentic consumers).** An **agentic** consumer **MUST NOT** let a manifest fetch — or any redirect hop, or any fetch of a manifest-supplied URL such as `logo` — reach an internal/loopback/link-local/RFC 1918/RFC 4193/metadata-service address. It **MUST** resolve the target host and validate the resolved IP against a deny-list **before each connection**, including re-validating **before each redirect hop** (defeat DNS-rebinding and redirect-to-internal). Non-agentic consumers SHOULD do the same.

### 7.10 Signature does not launder authority

Restating [§6](#6-signing-and-integrity) as a security rule: a valid signature proves authorship by an out-of-band-trusted key and detects tampering; it does **not** make any field authoritative. A signed manifest asserting deductibility is exactly as forbidden as an unsigned one. Consumers MUST NOT relax any [§7.2](#72-the-three-forbidden-authorities) check because a manifest is signed.

### 7.11 Denial-of-service and abuse

A publisher may serve a huge, deeply-nested, or slow manifest. Beyond the size/timeout caps above, consumers SHOULD bound nesting depth and total key count and SHOULD rate-limit manifest fetches per origin. A malformed manifest MUST fail safe (no partial authority extraction).

---

## 8. Conformance: strict vs. tolerant

A conforming consumer implements at least one of two modes; the reference parser implements both:

- **Strict (`Parse`).** Rejects any unknown field (`additionalProperties: false` / `DisallowUnknownFields`), rejects trailing data, rejects authority-named keys in `x_extensions`, and validates all constraints. Fails closed. RECOMMENDED for ingestion pipelines and `givmo manifest validate`.
- **Tolerant (`Sanitize`).** Accepts the well-formed core and **drops** every unknown or authority-asserting field: an unknown field never lands on the typed result, and an authority-named `x_extensions` key is **deleted from the returned extension bag** — every dropped field is returned as a `RejectedClaim`, plus non-fatal `Warnings`. (An authority-named key nested deeper *inside* a retained extension value is reported for telemetry; the value itself remains opaque, display-only data under [§7.3](#73-the-consumer-behavior-guarantee-normative).) It also **blanks any invalid URL it would otherwise keep** — a `website`/`logo`/`self_hosted_url`/`support_url` that is not a valid `https` URL is dropped (set empty) and reported, so a downstream caller never receives a non-`https` or malformed URL as if it were valid; other constraint violations on retained fields (e.g. an over-long `description`) are surfaced as `Warnings` for the caller to enforce at use time. Never fails on overreach; returns no manifest only if the mandatory core is unparseable. Appropriate for best-effort discovery where the goal is to salvage benign metadata while telemetering hostility.

In **both** modes the guarantee is the same and it is the **consumer-behavior guarantee of [§7.3](#73-the-consumer-behavior-guarantee-normative)** — no manifest field is mapped to a payee/eligibility, deductibility, or receipt/legal-copy decision. The reference parser additionally realizes this via the unrepresentable-authority technique ([§7.4](#74-implementation-technique-unrepresentable-authority-non-normative)), but a consumer built from the schema alone MUST enforce [§7.3](#73-the-consumer-behavior-guarantee-normative) directly. Because tolerant mode blanks invalid URLs, callers still MUST re-validate any URL's scheme/host at use time ([§7.9](#79-fetch-time-hardening)) — never trust that the producer honored the pattern.

---

## 9. Versioning

- `manifest_version` is SemVer. This document defines the `1.x` line.
- A consumer MUST reject a **major** version it does not implement (a `2.x` manifest is not processed by a v1-only consumer). The schema's `manifest_version` pattern (`^1\.[0-9]+(\.[0-9]+)?$`) is **v1-specific by design** — it accepts only the `1.x` line this document defines; a future major version ships its own schema with its own pattern, and a `2.x` manifest is expected to *fail* this v1 schema, which is the intended major-version rejection.
- Within the `1.x` line, changes are **additive and backward-compatible**: new OPTIONAL fields only. A v1.0 consumer encountering a v1.1 manifest processes the fields it knows and ignores/drops the rest (tolerant mode) or rejects unknown fields (strict mode) — publishers targeting broad compatibility SHOULD emit the lowest `1.x` that carries their data.
- Security-relevant constraints (the three forbidden authorities, `alg` allowlist, `https`-only) are invariant across the `1.x` line and any field added in a minor version MUST NOT be able to assert authority.
- The `x_extensions` bag is the escape valve for experimentation without a version bump; extension keys never carry authority.

---

## 10. IANA Considerations

> **Registration status (updated 2026-07-28).** An initial provisional registration request (submitted 2026-07-24) was declined by the registry's designated expert under current registry policy: generic ("bare-word") URI suffixes are accepted only when sourced from an IESG-recognised standards organisation, and a single-owner GitHub repository is not a suitable specification reference. Rather than adopt a vendor-prefixed suffix — which would defeat the manifest's purpose as a platform-neutral discovery location — the editor is pursuing the path the reviewer suggested: an Internet-Draft and a proposal through the IETF DISPATCH process. Until a registration is granted, `/.well-known/donate.json` operates as an unregistered well-known URI. The template below is retained as the registration this specification requests; this section will be updated as the registration's status changes.

Registration template (per RFC 8615 §3.1):

- **URI suffix:** `donate.json`
- **Change controller:** Givmo Charitable Fund (`givmocharitable.org`); Givfi, Inc. engineering maintains the reference implementation and repository under GCF's governance.
- **Specification document(s):** This document (the `donate.json` Manifest Specification v1.0), published in the specification repository.
- **Status:** provisional
- **Related information:** The resource is a JSON document (`application/json`) describing an organization as a charitable-giving discovery target. It is untrusted, attacker-writable input and is not authoritative for recipient eligibility, tax-deductibility, or legal/receipt copy. See the Security Considerations of the specification.

No new media type is registered by this document; the resource uses `application/json`. No other IANA registries are affected. (If a dedicated media type such as `application/donate+json` is desired later, that registration is a separate, future action and is out of scope for v1.0.)

---

## 11. Legal model (normative constraints on consumers)

For the reference consuming platform, and for any consumer operating a donor-advised-fund or similar intermediated-giving model, the following constrain how a manifest may be used. They are stated as design constraints and the tax-law rationale for them; they are **not** legal or tax advice.

- **Donee model.** A `mediated` gift is an irrevocable gift to the consuming platform's charitable entity (for Givmo: Givmo Charitable Fund, a 501(c)(3) DAF sponsor), which takes exclusive legal control and grants onward at its discretion on the donor's nonbinding recommendation; a recommended organization **may not receive the funds**. A manifest MUST NOT be used to represent a gift as a direct, deductible gift to the named organization. The manifest cannot, via this schema, make any such representation.
- **Deductibility is platform-communicated and DAF-limited.** Whether and how much a gift is deductible is communicated **only** through the consuming platform's authoritative copy — never stated by or inferred from the manifest ([§7.2](#72-the-three-forbidden-authorities)). Deductibility is subject to IRS limitations, **including DAF-specific limits**: contributions to a donor-advised fund are **excluded from the federal non-itemizer ("universal") charitable deduction** and are subject to the applicable **itemizer floor (the 0.5%-of-AGI floor for tax years beginning after 2025-12-31)** under the 2025 OBBBA changes. A consumer MUST NOT present any manifest-derived content as an unqualified "tax-deductible" or "100%" claim; qualified, platform-authored copy governs.
- **Recipient integrity as a control, not a deduction guarantee.** Routing a grant to a non-qualified or fraudulent recipient is framed as a **§ 4966 taxable-distribution and operational-control risk** for the DAF sponsor, and defeats the accuracy of the platform's disclosures. (This specification does **not** claim that a bad downstream grant *automatically* defeats a donor's deduction; absent a conduit/earmarking fact the donor's deductible event is the completed gift to the sponsor.) The control response is the authoritative eligibility resolution of [§7.2](#72-the-three-forbidden-authorities), re-confirmed at settlement, with an eligible alternate selected if a recommended recipient fails.
- **An agent is never the donor (design invariant + rationale).** As a **design invariant**, every gift resolves to a real, terms-accepted human donor who pays and accepts terms on the consuming platform's own hosted page; the agent never accepts terms, never holds a card or secret, and is always identified as an agent. The tax-law reason a human donor is required: the charitable deduction and DAF advisory privileges attach to a *donor* — a person who makes a completed gift and (for a DAF) holds nonbinding advisory privileges — and the required contemporaneous written acknowledgment, including the DAF exclusive-legal-control affirmation, is the donor's. (This is stated as the reason a human-donor design is required; it is **not** asserted here as settled law on the legal or contractual capacity of AI agents.)
- **Regulated, not exempt.** The consuming platform operates a regulated charitable-fundraising posture (e.g. California AB 488; Hawaii Act 205 from 2026-07-01, with by-name charity use consent- or geo-gated there). A manifest MUST NOT be used to hold the program out as unregulated or exempt, and the platform's jurisdiction-appropriate disclosures are selected server-side and bound per transaction — never sourced from the manifest.

These constraints exist so that even a perfectly-formed, signed manifest from a real charity cannot cause the consumer to make a deductibility claim, issue a receipt, or move money in a way the manifest — untrusted input — dictates.

---

## 12. Relationship to consuming platforms (non-normative)

The specification is **open**: any site may publish a conforming manifest, and any consumer may process one. This is intentionally decoupled from **catalog acceptance**. Whether a given manifest results in the organization appearing in, or being transactable through, a particular platform's catalog is that platform's **curated** decision, governed by its own onboarding, screening, and authoritative-data resolution — not by the manifest and not by this specification. For Givmo specifically: publishing `donate.json` does not enroll an organization in Givmo's catalog; Givmo continues to curate which organizations it surfaces and mediates, and it resolves identity and eligibility against authoritative IRS data regardless of manifest content.

---

## Appendix A — Examples

### A.1 Minimal valid manifest

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

### A.2 Fuller valid manifest

See `examples/valid.donate.json`. It exercises display metadata, contact info, aliases, and `method: mediated`, and validates cleanly against the schema (0 errors).

### A.3 Hostile manifest (rejected / sanitized)

See `examples/hostile.donate.json`. It attempts every forbidden move: prompt-injection payloads embedded in `legal_name` and `description`; self-asserted `tax_deductible`, `is_qualified_501c3`, `good_standing`, `pub78_verified`, `receipt_text`, `legal_disclosure`; a `self_hosted` method with an exfiltration `self_hosted_url` plus rogue `action_url` / `payto` / `confirmed_recipient` / `skip_verification` fields; and `x_extensions` keys (`givmo:catalog_accepted`, `givmo:eligibility_override`, `authorization`) trying to spoof catalog membership and carry a bearer token.

Against this manifest a conforming consumer:

- **Strict mode:** REJECTS the manifest (unknown/authority fields present).
- **Tolerant mode:** salvages the display-safe subset: `manifest_version`; `organization.legal_name`, `.country`, `.ein`-as-key, `.website`, and `.description`; `donation.method` (`self_hosted`, carried honestly) together with `donation.self_hosted_url`; and `display.categories`. The `self_hosted_url` here is the attacker's exfiltration URL — it is a *valid* `https` URL, so tolerant mode retains it, and that is safe **only** because it survives strictly as **non-actionable display data** under [§7.6](#76-no-manifest-authored-action-urls): an agentic consumer never auto-navigates to it, never passes it to a model as a tool target, never attaches a credential to it, and hands it (if at all) to a human out of band. Everything else is removed: every deductibility/eligibility/receipt/legal field and every rogue `donation` key is **dropped** as an unknown field, all three authority-named `x_extensions` keys are **deleted from the returned extension bag**, and each dropped field is reported as a `RejectedClaim` for telemetry. The injection strings in `legal_name`/`description` survive only as untrusted, provenance-labeled display text and never reach an instruction position (markup-stripping is the consumer's display duty under [§7.7](#77-prompt-injection-and-untrusted-text-handling)). No authoritative action target, no deductibility claim, no receipt, and no eligibility assertion is ever extracted. The `ein` (`00-0000000`) is still only a *key*, and would fail authoritative IRS resolution, so the record is display-only with no donation affordance ([§5.3a](#53a-identity-binding-normative)).

### A.4 Hostile manifest — value-side and scan-bypass (`hostile-value-side.donate.json`)

This fixture asserts authority via **values** under innocuous keys and via `x_extensions` keys the name-scan does **not** enumerate (`beneficiary_payout`, `vendor:routing_hint`, a `note` value carrying "eligible=true; deductible=true; receipt=…"), plus a non-`https` `logo`. It exists to prove the guarantee does **not** rest on the scan — or on strictness: the fixture deliberately contains **no unknown fields at all** (every key is spec-defined or lives inside the open `x_extensions` bag), and its extension keys evade the name-scan, yet the manifest is still safe because (a) strict mode rejects it anyway — solely on the invalid (non-`https`) `logo`, with neither unknown-field rejection nor the scan contributing; (b) in tolerant mode the hostile content survives only inside the **opaque `x_extensions` map** as untrusted display data, is never mapped to any authority decision ([§7.3](#73-the-consumer-behavior-guarantee-normative)), and the non-`https` `logo` is **blanked and reported** ([§8](#8-conformance-strict-vs-tolerant)); and (c) nothing lands on a typed authority field because none exists ([§7.4](#74-implementation-technique-unrepresentable-authority-non-normative)). A clean name-scan here would still be safe — that is the point. (And had the `logo` been a well-formed `https` URL, strict mode would have *accepted* the manifest — with the hostile values still inert for exactly the reasons in (b) and (c): the guarantee rests on the consumer rule, not on rejection.)

---

## Appendix B — Reference artifacts

- **JSON Schema:** `schema/donate.schema.json` (draft 2020-12).
- **Reference parser:** `parser/donate.go` (Go, standard-library only) + `parser/donate_test.go`; `parser/go.mod` (no external deps).
- **Examples / test fixtures:** `examples/valid.donate.json`, `examples/hostile.donate.json`, `examples/hostile-value-side.donate.json`.

---

## Appendix C — References

**Normative.**

- RFC 2119 / RFC 8174 — Requirement-level keywords.
- RFC 8615 — Well-Known Uniform Resource Identifiers.
- RFC 7515 — JSON Web Signature (JWS) — detached form (§A.5).
- RFC 3339 — Date and Time on the Internet: Timestamps.
- ISO 3166-1 — Country codes (alpha-2).
- BCP 47 — Tags for Identifying Languages.
- JSON Schema draft 2020-12.

**Informative.**

- IRS Publication 78 / Tax-Exempt Organization Search (TEOS); IRS Business Master File (BMF); IRS Automatic Revocation of Exemption List — the authoritative recipient-eligibility sources referenced by [§7.2](#72-the-three-forbidden-authorities).
- RFC 8785 — JSON Canonicalization Scheme (JCS). *Considered and deliberately NOT adopted* for signing; this spec uses textual excision instead ([§6.2](#62-signing-payload--textual-excision-no-canonicalization)).
- RFC 1918 / RFC 4193 — private IPv4 / unique-local IPv6 address ranges, referenced by the SSRF guard ([§7.9](#79-fetch-time-hardening)).
- OBBBA (2025) DAF deductibility changes — non-itemizer exclusion and the 0.5%-of-AGI itemizer floor, referenced by [§11](#11-legal-model-normative-constraints-on-consumers). (Informative pointer; not tax advice.)
