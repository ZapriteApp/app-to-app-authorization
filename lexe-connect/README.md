# Lexe Connect

Working documents for the same-device app-to-app delegated authorization flow between Zaprite P2P and Lexe.

The architecture and protocol are under discussion. This repository is private while the teams develop the proposal.

## Current discussion

- [Zaprite architectural proposal and discussion](https://gist.github.com/nk1tz/b459fb8804612c34f6a6ff201a62ec6c)
- [Lexe connection-flow draft](https://gist.github.com/MaxFangX/5ed4d545c1368b4fa2ba06ed3200f68f)

The architectural gist's comments discuss changes to its original recommendations. Read the discussion alongside the proposal; the body alone does not reflect all subsequent feedback.

## Document plan

- `architecture.md`: the approach, trust assumptions, responsibilities, and open architectural questions.
- `protocol.md`: exact messages, fields, cryptographic parameters, validation rules, and errors.
- `examples/`: example requests and responses, followed by shared test vectors.

These documents will be added as the work progresses. Propose changes through pull requests so both teams can discuss specific lines and record agreed decisions.
