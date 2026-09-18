# Lexe Connect architecture

Status: working proposal for joint review. Discussion baseline: September 18, 2026.

This document describes a same-device connection flow. A user taps Connect Spending in Zaprite P2P, approves access in Lexe on the same phone, and returns to Zaprite with a usable credential. Lexe continues to issue and enforce its own credentials.

The current direction uses verified app-only links and a confidential random state value. It does not require either company to sign connection messages or exchange protocol signing keys. HPKE remains optional protection for the returned credential. This revises the signing-based recommendations in the original architectural gist.

The [protocol draft](protocol.md) translates this approach into candidate message and validation rules. Neither document records final joint approval. Open decisions must be resolved before treating the protocol as an interoperable specification.

## Decision status

| Challenge | Current direction | Evidence and remaining decision |
| --- | --- | --- |
| Identify the requesting integration | Obtain branding from the callback domain, or display the domain itself. No backend-signed request. | Max proposed this; Nate accepted the direction. This identifies the destination and does not prove who launched the request. |
| Open genuine Lexe | Open `https://lexe.app/connect` using the verified app association. | Both sides support HTTPS app links. Nate additionally requires app-only delivery with no fallback; confirm that requirement and platform behavior with Lexe. |
| Obtain user consent | Approval grants the full requested scope set; otherwise the attempt fails. | Max explicitly confirmed this. Exact scopes and budget behavior remain to be defined. |
| Deliver the credential | Return to the verified Zaprite app on the same device. Offer optional HPKE encryption. | Both sides support the direction. Confirm strict app-only callback behavior and the response encoding. |
| Match the result to the request | Echo a fresh, unpredictable secret in `metadata`; accept only a matching pending attempt. | Max proposed this; Nate accepted it. Confidentiality, expiry, repeated-use handling, and result placement need specification. |
| Obtain receiving information | Associate the receiving address and wallet identity with the approved credential. | Open: include them in the callback or retrieve them through an authenticated Lexe API. Availability through `client-info` has not been confirmed. |

Sources: [Max's response](https://gist.github.com/nk1tz/b459fb8804612c34f6a6ff201a62ec6c#gistcomment-6377889), [Nate's reply](https://gist.github.com/nk1tz/b459fb8804612c34f6a6ff201a62ec6c#gistcomment-6377918), and [Lexe's connection draft](https://gist.github.com/MaxFangX/5ed4d545c1368b4fa2ba06ed3200f68f).

## Scope

The initial integration is native Zaprite P2P and native Lexe on one phone. It covers authorization and connection setup.

Browser-based setup, cross-device QR pairing, and `post_url` delivery to a server are separate integration modes. The confidentiality assumptions in this document do not automatically apply to them. Lexe can support those modes in its broader proposal without this integration implementing them.

Payment execution, ongoing budget-exhaustion errors, credential revocation, and credential expiry belong to Lexe's wallet API contract. Setup must report the granted access accurately, but does not define that payment API.

## Trust and infrastructure

Both apps trust the phone's supported operating system to enforce domain-to-app associations. Each receiving app maintains its HTTPS association files and matching native app configuration. Opening an HTTPS URL alone is insufficient; the sender must enforce the intended verified app destination and stop if it cannot do so.

Zaprite's callback domain is also the source of its display identity. Lexe may fetch a well-known branding document from that domain. The final path and branding lookup rules are open. A supplied name, icon, package name, or callback string alone is not proof of ownership.

The random state value is confidential authentication material for one attempt. Zaprite sends it only through the verified Lexe handoff and accepts it only for the pending Lexe connection. Its secrecy is a central assumption of removing response signatures. A matching value does not independently sign or certify the rest of the result.

The design trusts both apps to handle credentials and state values correctly. It does not defend against a compromised operating system or either endpoint leaking plaintext. OS routing and diagnostics are distinct concerns: verified delivery does not promise that every diagnostic path redacts every URL.

Lexe has stated in the design discussion that its ordinary servers should not learn the spending credential. This flow does not require a company-operated server to receive that plaintext. Lexe must confirm how credential issuance fits its existing app and trusted execution environment.

## Connection flow

1. Zaprite creates a pending connection attempt with fresh secret state, its expected callback, requested access, and a local deadline. If encryption is requested, it also creates a temporary HPKE keypair and retains the private key locally.
2. Zaprite opens Lexe's verified HTTPS entry point with the request. Failed verified delivery ends the attempt; any installation guidance is a separate action without the sensitive request URL.
3. Lexe validates the request and callback destination, obtains the destination's display identity, and presents the requested access to the user.
4. The user approves all requested scopes or rejects the request. Lexe creates the credential after approval and applies the approved policy.
5. Lexe returns the result and matching state to Zaprite through verified app-only delivery. If Zaprite requested HPKE, Lexe encrypts the result according to the agreed protocol.
6. Zaprite validates and matches the response, decrypts when applicable, and checks the credential through the trusted Lexe SDK/API as needed. Max identified `client-info` as a way to inspect granted scopes.
7. Zaprite commits the accepted connection and consumes the pending attempt once. Spending credentials remain local; authenticated receiving information may be sent to the Zaprite API.

Receiving information is a completion dependency only for the receiving feature. The teams must decide whether it is required for this combined setup or can be obtained separately without blocking Connect Spending.

## Security properties and limits

| Mechanism | What it provides | What it does not provide |
| --- | --- | --- |
| Verified app-only opening | Selects the intended associated app and prevents browser fallback when enforced correctly. | Proof of which app initiated an incoming link, or a guarantee against diagnostic recording. |
| Domain-hosted branding | Attributes display metadata to the callback domain. | Evidence that Zaprite's backend approved this particular request. |
| Confidential random state | Rejects forged responses from an attacker who cannot learn or guess the pending secret, and supports request correlation. | A signature over the payload; protection after the secret is exposed. |
| Optional HPKE | Keeps the returned credential confidential if the encrypted callback is exposed and the device private key remains secret; detects alteration of the encrypted message. | Sender authentication in Base mode, protection of the outbound state, or protection against plaintext logging at an endpoint. |
| Credential inspection | Checks what the credential actually permits through Lexe's trusted API. | By itself, proof that the credential belongs to the wallet the user just approved. State and delivery assumptions still matter. |

The design permits an arbitrary app to attempt to initiate a request naming Zaprite's destination. Lexe still requires user consent, delivers only to the verified destination, and Zaprite rejects unsolicited results. The architecture does not promise to eliminate nuisance requests or prove backend endorsement.

Response acceptance must also check the expected connection context. A valid credential for an attacker's wallet is not sufficient. A public receiving address needs integrity protection because substitution could redirect future payments.

Random state matching follows established [native-app request-forgery guidance](https://www.rfc-editor.org/rfc/rfc8252.html#section-8.9). This custom flow is not thereby a complete OAuth implementation. HPKE's [security properties](https://www.rfc-editor.org/rfc/rfc9180.html#section-9.1) distinguish encryption from sender authentication.

## Consent and budgets

Zaprite requests only the scopes necessary for the feature. Lexe requires approval of the full requested set; declining required access produces no successful connection. Zaprite can double-check the actual scopes after issuance.

Budget choice is separate from scope approval. The teams must define whether a requested budget is an exact requirement, a protective maximum, or a suggestion. An omitted or unsupported budget must not silently turn a required spending limit into unlimited access. Lexe enforces the final policy; Zaprite does not maintain an authoritative parallel spending counter.

Budget schema, supported limits, units, fees, renewal, and inspection remain open. This document does not assume that a budget capability is already implemented.

## Ownership and interruption

Zaprite owns local pending state, authentication to its own services, any app-attestation policy, secure credential storage, and the connection UI. Spending credentials never sync to other devices, including in encrypted form. Public receiving information can be stored by the Zaprite API after its association with the connection has been validated.

Lexe owns consent, credential issuance, and enforcement. Both teams own their domain associations and safe URL handling.

A user can restart a failed attempt with fresh state and, if used, a fresh encryption keypair. Recovery does not require a receipt-acknowledgment endpoint. Successfully opening the callback is not proof that Zaprite accepted or stored the credential. Lexe's treatment of an issued credential after known delivery failure remains an explicit decision; this draft does not require durable resumption or automatic revocation.

## Next decisions

The [protocol decision register](protocol.md#open-decision-register) tracks the exact questions for joint review. The architectural questions to resolve first are:

- Confirm strict verified app-only delivery on supported iOS and Android versions.
- Confirm the assumption that secret state can remain confidential throughout the intended handoff.
- Choose the receiving-information mechanism and its relationship to Connect Spending completion.
- Define the budget policy Lexe can offer and enforce.

The protocol draft proposes fragment-based return values, bounded input parsing, expiry, and duplicate handling. These details are candidates for agreement, not claims that Lexe has already accepted them.
