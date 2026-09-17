# FAILURE-MODES.md — Failure catalog for payments-svc

This file records failure modes observed when using AI to generate tests and
review code in `payments-svc`.

## Category: Test generation

- [n1] The AI generates only happy-path tests. It omits the `amount == 0`
  boundary and equivalence classes. Tests pass without verifying meaningful
  behavior.

## Category: Code review

- [n1] The AI comments on style (names, spacing) and misses critical issues:
  missing authorization and refunds exceeding the original payment amount.
