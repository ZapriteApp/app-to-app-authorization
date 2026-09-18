# Lexe Connect protocol draft

Status: candidate specification for joint review. Discussion baseline: September 18, 2026.

This document develops the [architecture](architecture.md) into protocol rules. It preserves Lexe's proposed field names where possible. Candidate requirements below are proposals, not an agreed or implementation-ready version 1. Items in the [decision register](#open-decision-register) block interoperability until resolved.

## 1. Profile and participants

This profile covers a requesting native app and the Lexe wallet app on the same device. The first integration is Zaprite P2P. Both use verified HTTPS app-only delivery. It excludes `post_url`, browser fallback, custom-scheme fallback, and cross-device setup.

There are no company message signatures in this draft. The operating system still verifies the platform's domain/app association. The confidential per-attempt state value is separate from any optional HPKE keypair and from the issued spending credential.

## 2. Request model

The proposed entry point is `https://lexe.app/connect`. Its precise production paths and supported protocol-version mechanism need confirmation.

| Field | Candidate rule for this profile | Decision still needed |
| --- | --- | --- |
| `scopes` | Required, nonempty set of requested Lexe scopes. All requested scopes require approval. | Exact vocabulary, list encoding, and minimum Connect Spending set. |
| `permissions` | Do not assume this is synonymous with `scopes`. | Lexe must define its independent meaning or remove it; requiredness is undecided. |
| `budget` | Express the requested spending policy when used. | Schema, requiredness, limits supported, and whether the request is exact, a maximum, or a suggestion. |
| `redirect_uri` | Required HTTPS callback associated with the requesting app. Preserve it in local pending state and validate it before use. | Allowed hosts and paths, port/query rules, normalization, and app-identity discovery. |
| `post_url` | Rejected in this profile. | Other Lexe profiles can define server delivery separately. |
| `hpke_pubkey` | Optional temporary device public key. Its presence requests encrypted success delivery; an invalid or unsupported key fails the attempt. | Suite, key type, encoding, and capability discovery. |
| `metadata` | Required for this profile and returned unchanged. Carries fresh confidential state for the pending attempt. | Opaque state string versus an encoded structured value containing state and application context. |
| `label` | Optional suggested connection label. Treat as display text. | Size, encoding, and how Lexe lets the user edit it. |
| `app_name`, `app_icon` | Incoming values are not trusted identity. Obtain trusted branding from the validated callback domain, or display that domain. | Whether to omit these request fields, ignore them, or retain them as explicitly unverified hints. |
| Protocol version | The messages need an unambiguous version. | URL path, explicit field, or another agreed mechanism. |
| Request identity and freshness | Associate each response with one bounded pending attempt. | Whether a separate public `request_id` and wire expiry are needed in addition to secret state and a local timeout. |

Lexe's gist uses `hpke_pubkey` and `redirect_uri` in its field list but `pubkey` and `redirect_url` in its example. Resolve this before implementation; this draft does not establish aliases.

Its HPKE public-key type also needs correction or explanation. Select the encryption/key-agreement key type required by the agreed HPKE suite, such as X25519, rather than treating `ed25519::PublicKey` as directly interchangeable. See [RFC 9180](https://www.rfc-editor.org/rfc/rfc9180.html#section-7.1).

## 3. Pending state and secret handling

Before opening Lexe, the requesting app records:

- Fresh state from a cryptographically secure random generator.
- The expected wallet provider, callback, feature, and destination slot in the app.
- Requested scopes and budget requirements.
- The expected response mode, plaintext or HPKE-encrypted.
- An expiry deadline and the attempt's completion state.
- The temporary private encryption key if HPKE was requested.

Proposed minimum state entropy is 128 bits. The encoding and attempt lifetime need joint agreement. State is not a predictable counter, account identifier, static installation identifier, or reusable credential.

Neither endpoint includes secret state or full sensitive handoff URLs in ordinary logs, analytics, or error reports. A user-facing correlation ID, if needed, is separate from secret state.

An attacker who learns state may forge a matching callback. If it also learns the public encryption key, HPKE Base mode does not prevent it from constructing its own encrypted result. Encryption of the return does not resolve an outbound state leak.

Accept a successful result at most once for an active attempt. Define request re-opening separately: opening the same request in Lexe again is not authorization to silently create another credential. The teams must choose whether Lexe rejects it, reuses an earlier result, or requires another approval, and how long any deduplication record lasts.

## 4. Verified delivery

The sender validates the HTTPS destination and invokes an app-only opening mechanism. Failure ends that handoff; do not retry the sensitive URL in a browser, custom scheme, clipboard, or alternate recipient.

On iOS, the receiving app configures associated domains and its domain publishes `apple-app-site-association`. The sender uses `UIApplication.open` with `universalLinksOnly: true` and handles failure without a fallback. See [Apple's opening option](https://developer.apple.com/documentation/uikit/uiapplication/openexternalurloptionskey/universallinksonly).

On Android, the receiving app uses verified App Links backed by `assetlinks.json`, its package name, and signing certificate. The sender must enforce the verified association to the expected recipient. A candidate implementation on Android 12+ checks domain verification, restricts the intent to the expected package, and prevents browser/chooser fallback. User selection of an unverified handler is not equivalent to domain verification. Non-browser flags alone do not authenticate the package. Package discovery, visibility declarations, certificate/association checks, and supported older versions require agreement and device testing. See [Android verification](https://developer.android.com/training/app-links/verify-applinks) and [Intent restrictions](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_REQUIRE_NON_BROWSER).

Successfully handing a link to an app does not prove that it accepted the connection. Avoid interpreting an OS opening callback as an application-level receipt.

## 5. Destination identity and consent

Lexe derives the display domain from the validated callback. A proposed well-known document supplies `appName` and `appIcon`; the path is not settled. Retrieve it over HTTPS from the callback's trusted origin. Define fetch limits and redirect rules so a different origin cannot silently become the identity authority. Missing branding falls back to the domain.

The approval screen explains the destination and requested scopes. Grant all requested scopes or reject. Do not silently grant additional scopes or treat an unsupported required scope as optional.

The budget contract must define requested versus granted values, units, treatment of fees, renewal, credential expiry, and omission behavior. If a requested protective limit cannot be enforced, do not silently issue unlimited spending authority.

## 6. Result model and callback format

Each response describes exactly one outcome: success, user rejection, or setup failure. It carries the state associated with that attempt. An error response contains no credential.

Lexe's current draft places `credential`, `credential_ciphertext`, or `error` in the callback query. This draft proposes placing result values in the callback fragment instead. That keeps result data out of ordinary destination HTTP requests if a URL is accidentally fetched; it does not make the fragment confidential to apps or system diagnostics. The teams must explicitly resolve this difference.

| Result information | Candidate requirement | Wire decision |
| --- | --- | --- |
| State | Return the exact state value associated with the request. | `metadata` encoding and placement in encrypted mode. |
| Outcome | Distinguish success from rejection and failure. | Explicit status field versus mutually exclusive payload fields. |
| Spending credential | Required for a successful spending connection. | Existing credential serialization and SDK ingestion contract. |
| Granted scopes | Zaprite must establish the actual granted scopes before accepting success. | Callback field or `client-info`; Max confirmed scope inspection through that method. |
| Approved budget and credential expiry | Establish the enforced policy when required by the request. | Callback fields or supported authenticated API; method availability is not yet confirmed. |
| Receiving address and wallet identity | Bind receiving configuration to the approved connection. | Callback versus authenticated lookup; availability through `client-info` is open. |
| Setup error | Distinguish rejection, unsupported request, invalid format, and other agreed failures. | Exact code vocabulary, retry hints, and representation. |
| Version and request binding | Interpret the result under the correct rules and pending attempt. | Exact fields and HPKE context binding. |

Do not accept a mixture of success and error payloads, both plaintext and ciphertext credential forms, or conflicting duplicate fields. Do not install receiving information from an unsolicited callback or an unrelated lookup result.

For a spending-only setup, decide whether receiving-information lookup can complete later. If obtaining receiving information requires a broader permission, request that permission explicitly with the appropriate user consent.

## 7. Optional HPKE

If `hpke_pubkey` is absent, the candidate profile permits plaintext delivery through the verified app-only callback. Both implementations must explicitly agree to support that mode. It retains the requirement to keep state and credentials confidential.

If `hpke_pubkey` is present, encryption is mandatory for the successful response to that attempt. Do not silently downgrade to plaintext if encryption is unsupported or fails. The requesting app rejects a plaintext success when its pending attempt requested encryption.

The proposed encrypted plaintext contains the credential and the state together with any returned wallet identity, receiving information, and approved policy. Any public outer request identifier is only a lookup aid, not authentication. Exact placement and authenticated context are open decisions.

The specification must select the HPKE mode, KEM, KDF, AEAD, public-key encoding, encapsulated-key and ciphertext representation, application context, and additional authenticated data. These must bind the encrypted result to the intended protocol and pending request. Include the granted scope set in the result; do not confuse it with requested scopes when defining the binding.

HPKE Base mode provides confidentiality and ciphertext integrity, not sender authentication. No company signing key is introduced by choosing HPKE. Encryption protects a captured encrypted callback only while the corresponding private key is confidential. It does not protect plaintext before encryption or after decryption.

Whether setup errors use encryption, and how unsupported encryption can be reported safely, remains to be specified. [RFC 9180](https://www.rfc-editor.org/rfc/rfc9180.html) supplies the cryptographic construction; this protocol must define its use and serialization.

## 8. Response acceptance

The following is a candidate acceptance procedure, not an agreed wire parser:

1. Validate the callback destination against the locally recorded callback and apply agreed size and parsing limits.
2. Identify a live pending Lexe attempt without treating an unverified identifier as proof of origin.
3. Enforce the expected response mode. For HPKE, decrypt with that attempt's local key and agreed context.
4. Validate the secret state and outcome; reject expired, unsolicited, conflicting, or already-completed responses.
5. Validate the credential representation and inspect actual scopes through the trusted Lexe SDK/API. Do not accept an arbitrary API endpoint supplied by the callback as the authority.
6. Check any required budget and expiry policy. Resolve receiving information through the agreed mechanism before enabling receiving or publishing it to the Zaprite API.
7. Commit the accepted connection and mark the attempt completed once. Prevent simultaneous callbacks from installing the connection twice. Discard temporary state and encryption key when complete.

Invalid callbacks must not overwrite an existing connection or consume another valid pending attempt. The result for a valid rejection or setup error closes the matching attempt under the agreed rules.

Exact state comparisons, byte encoding, callback normalization, parser behavior, and size limits are shared protocol decisions. UI screens, storage technology, and biometric controls remain local implementation choices.

## 9. Failure behavior

| Condition | Candidate behavior |
| --- | --- |
| Lexe cannot be opened through verified delivery | Stop locally; allow the user to retry with a fresh attempt. |
| Callback destination cannot be validated | Do not issue or send a credential to it. Show a local failure; do not blindly open it to report an error. |
| User declines the requested scopes | Return the matching rejection through the validated callback when possible. |
| Required scope or budget cannot be supported | Return a setup failure through a validated callback; do not silently broaden or weaken access. |
| State is missing, wrong, expired, or already consumed | Reject the callback without installing credentials or receiving information. |
| Requested encryption cannot be performed | Fail without sending a plaintext credential. |
| Decryption or payload validation fails | Reject the result without changing the existing wallet connection. |
| Temporary key or pending state is lost | Reject an unmatchable return and let the user start again. |
| Callback cannot be delivered after issuance | Report failure locally in Lexe; credential cleanup/revocation policy needs agreement. |
| Credential scope inspection fails | Do not mark setup successful; the exact retry and cancellation behavior needs agreement. |

No receipt acknowledgment is required in this draft. A timeout on Zaprite does not prove that Lexe never issued a credential, nor that Lexe revoked it.

## 10. Parsing and validation candidates

- Use an agreed UTF-8 and percent-encoding convention; define list and binary encodings explicitly.
- Reject duplicate single-valued parameters, conflicting outcome fields, and malformed encoding instead of selecting an arbitrary value.
- Set shared limits for URLs, metadata, labels, scopes, icons, and encrypted responses.
- Specify how unknown fields and versions behave. Do not silently ignore an unknown security requirement.
- Validate callback scheme, origin, and path before delivery; define how existing query parameters and fragments are handled.
- Validate all externally supplied display text and URLs. Branding cannot override the validated destination identity.

These rules require concrete values and examples before interoperable implementations can be built.

## Open decision register

| ID | Decision | Who needs to resolve it |
| --- | --- | --- |
| D1 | Confirm verified app-only delivery, trusted recipient discovery, and supported OS versions. | Both teams, with device evidence. |
| D2 | Define `scopes` versus `permissions`, exact vocabulary, and the minimum requested set. | Lexe defines capabilities; Zaprite defines feature needs. |
| D3 | Define budget schema, enforcement availability, requested/granted semantics, and inspection. | Lexe proposes supported policy; both agree behavior. |
| D4 | Select callback response location and encoding, including query versus fragment. | Both teams. |
| D5 | Define state encoding, minimum entropy, expiry, request IDs, and repeated-request semantics. | Both teams. |
| D6 | Select the optional HPKE suite, key types, result envelope, context binding, and error behavior. | Both teams. |
| D7 | Choose callback or API retrieval for receiving address and wallet identity, with any required scopes. | Lexe confirms API capabilities; both agree the result contract. |
| D8 | Select well-known branding path and rules for callback origin, fetches, and redirects. | Both teams. |
| D9 | Define protocol versioning, error codes, parser rules, and size limits. | Both teams. |
| D10 | Define issued-credential behavior after delivery failure and how to avoid accidental duplicate connections. | Lexe proposes issuance behavior; both agree observable outcomes. |

The [examples and conformance cases](examples/README.md) provide a review checklist for these decisions. Add executable shared vectors after the formats and cryptographic parameters are settled.

## References

- [Lexe's original request-field proposal](https://gist.github.com/MaxFangX/5ed4d545c1368b4fa2ba06ed3200f68f)
- [Max's architectural response](https://gist.github.com/nk1tz/b459fb8804612c34f6a6ff201a62ec6c#gistcomment-6377889)
- [Nate's reply and proposed conditions](https://gist.github.com/nk1tz/b459fb8804612c34f6a6ff201a62ec6c#gistcomment-6377918)
- [NWA/NWC Wake hardening revision](https://github.com/ntheile/nwc-wake-spec/commit/ff51a68c79835423d7a208f361ab6dba61f8797c): related app-only handoff work, not a dependency or adopted specification.
- [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252.html): native-app authorization and request-state protections.
- [RFC 9180](https://www.rfc-editor.org/rfc/rfc9180.html): HPKE.
