# Contributing

Contributions to the Architect's Review Protocol are welcome. This document explains how to get involved.

## Ways to Contribute

- **Protocol improvements** — suggest changes to the review tiers, risk scoring, or phase structure
- **Language support** — add mapper examples for languages not yet covered
- **Bug reports** — if a protocol step produces unclear or incorrect results
- **Documentation** — improve clarity, fix typos, add examples

## How to Submit Changes

1. Fork the repository
2. Create a feature branch (`git checkout -b improve-tier-scoring`)
3. Make your changes
4. Commit with a clear message explaining the *why*
5. Open a pull request against `main`

## Guidelines

- **Keep protocols concise.** The architect's time is the constraint. Don't add steps that don't earn their cost.
- **Preserve tier calibration.** If you change what qualifies as CRITICAL vs MEDIUM vs LOW, explain your reasoning in the PR description.
- **Test with real codebases.** If you're changing the mapper or reviewer protocol, run it against a project and include the results in your PR.
- **One change per PR.** Don't bundle protocol changes with formatting fixes.

## Reporting Issues

Open a GitHub issue with:
- What you expected to happen
- What actually happened
- The approximate size and language of the codebase you reviewed (if applicable)

## Code of Conduct

Be respectful and constructive. This is a small project — keep discussions focused on making the protocol better.
