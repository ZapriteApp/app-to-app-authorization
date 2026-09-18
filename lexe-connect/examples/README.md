# Illustrative exchanges and conformance cases

These examples explain the candidate flow. They are not finalized wire messages, valid wallet credentials, or cryptographic test vectors. Property names in the JSON below describe logical information; they do not settle the protocol's field names or encoding.

Use the [protocol draft](../protocol.md) and its decision register when converting these into executable fixtures. All callback domains, state strings, credentials, and keys below are illustrative.

## Plaintext success

The requesting app records a pending attempt. Production state must come from a cryptographically secure random generator; the marker below is deliberately not a valid production value.

```json
{
  "request": {
    "wallet_entry_point": "https://lexe.app/connect",
    "redirect_uri": "https://client.example/lexe/callback",
    "metadata": "EXAMPLE_ONLY_FRESH_SECRET_STATE",
    "scopes": ["EXAMPLE_REQUIRED_SCOPE"],
    "encryption_requested": false
  },
  "returned_result": {
    "outcome": "success",
    "metadata": "EXAMPLE_ONLY_FRESH_SECRET_STATE",
    "credential": "EXAMPLE_ONLY_NOT_A_REAL_CREDENTIAL"
  },
  "scope_inspection": {
    "required_scopes_present": true
  }
}
```

The app validates the callback, matching active state, credential shape, and required access. It then installs the connection once and consumes the pending attempt. Receiving configuration is handled through the mechanism still to be agreed under D7.

## Encrypted success

The request includes a temporary public key. The private key stays on the requesting device. A candidate encrypted result contains:

```json
{
  "outcome": "success",
  "metadata": "EXAMPLE_ONLY_FRESH_SECRET_STATE",
  "credential": "EXAMPLE_ONLY_NOT_A_REAL_CREDENTIAL"
}
```

Lexe encrypts that result under the supplied public key using the agreed HPKE suite and context. It returns the envelope through the verified callback. The requesting app decrypts, matches state, and performs the same acceptance checks as above. The actual encapsulated key, ciphertext, additional data, and serialization are intentionally absent until D6 is resolved.

Returning a plaintext credential for this attempt is a failure, even if the secret state matches.

## User rejection

```json
{
  "outcome": "rejected",
  "metadata": "EXAMPLE_ONLY_FRESH_SECRET_STATE"
}
```

The result contains no credential. A matching active attempt moves to the agreed rejected state. The exact status/error code and encryption behavior remain open. An unsolicited rejection must not cancel an unrelated attempt.

## Candidate conformance cases

| Case | Expected behavior under this draft |
| --- | --- |
| Genuine apps installed with valid associations | Request and callback open their intended apps directly. |
| Wallet absent or outgoing association invalid | No browser/custom-scheme fallback; no sensitive link sent to another handler. |
| Requesting app absent or callback association invalid | No fallback delivery of the result. |
| Another app registers the same custom scheme | It receives no handoff from this profile. |
| Android user selects an unverified handler | User selection alone does not satisfy verification. |
| Callback state is missing or incorrect | No connection installed. |
| Callback has expired state | No connection installed. |
| Same successful callback delivered twice or concurrently | At most one accepted connection update. |
| Two simultaneous attempts receive out-of-order callbacks | Each result is matched to its own saved context; neither replaces the wrong connection. |
| Required scope declined | No successful credential grant for that request. |
| Credential lacks a required scope on inspection | Setup is not marked successful. |
| Required protective budget unsupported | Failure instead of silently unlimited access. |
| HPKE requested but plaintext success returned | Reject the result. |
| Encrypted payload altered or wrong decryption key used | Reject the result. |
| Duplicate state or conflicting success/error fields | Reject the malformed result. |
| Unexpected callback path or provider | Reject even if another pending attempt exists. |
| Callback contains attacker-chosen receiving information with no matching state | Do not install or upload that receiving information. |
| App restarts with no recoverable pending attempt | Reject the unmatchable callback; permit a fresh user-initiated attempt. |
| Callback delivery fails after issuance | Follow the credential cleanup policy agreed under D10; do not infer receipt. |

## Diagnostic checks

Use disposable marker strings to test release builds on supported devices. Inspect app logs and available system diagnostics around successful and failed handoffs. Record whether URL queries or fragments appear, whether they are redacted, and how access to the diagnostic output is restricted. Do not use live credentials for this exercise.

In encrypted mode, check that any captured callback contains ciphertext rather than plaintext credential material. Separately inspect outbound state handling. An absence of observed leakage on tested devices is useful evidence, not a universal guarantee about every OS or diagnostic configuration.

After the wire format is settled, add exact encoded URLs, parsing fixtures, and HPKE vectors with explicitly disposable keys. Both implementations should consume the same fixtures and agree on the outcomes.
