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

## amounts.py failure-mode review

Reviewed `src/payments_svc/amounts.py`, its callers, and repository documentation
against branch `prep/modulo-0-base-funcional` at commit `de0fa8d`. No tests were
written or run; observations below come from reading the code.

These are potential failure modes, not all confirmed defects. No documented
business policy establishes the correct limits, fees, or rounding rules.
"Input that rules it out" means candidate inputs for assessing the risk against
an agreed contract; an input alone cannot establish that contract. A confirmed
status identifies a supported contract, not proof that every implementation
path satisfies it.

### Boundary

#### B01 — Lower limit and zero

- ID: B01
- Category: Boundary — lower limit and zero
- Risk: Zero-value payments or tiny positive payments receive unintended treatment.
- Input that rules it out: `Decimal("-0.01")`, `Decimal("-0.00")`, `Decimal("0")`, and `Decimal("0.001")`, with `"USD"`.
- Current behavior observed in the code: Negative amounts are rejected. Both signed and unsigned zero pass validation and receive zero fees. Any positive amount attracts at least the minimum fee.
- Recommended expected contract: Decide whether payments must be strictly positive, whether signed zero is canonicalized, and the smallest permitted payment.
- Contract status: pending decision
- Why it matters for payments: A tiny positive amount can trigger a fee much larger than its principal; zero transactions may have different operational meaning.

#### B02 — Maximum amount

- ID: B02
- Category: Boundary — maximum amount
- Risk: The limit is applied at the wrong threshold or to the wrong monetary component.
- Input that rules it out: `Decimal("99999.99")`, `Decimal("100000.00")`, and `Decimal("100000.01")`, with `"USD"`.
- Current behavior observed in the code: Exactly `100000.00` passes; larger principal amounts fail. The same limit applies to every currency. At the maximum USD principal, the calculated total is `102900.00`.
- Recommended expected contract: Specify inclusivity, limits by currency, and whether the cap covers principal, fee, total, or multiple components.
- Contract status: pending decision
- Why it matters for payments: A principal-only check may permit a final debit above an intended transaction limit.

#### B03 — Minimum-fee crossover

- ID: B03
- Category: Boundary — minimum-fee crossover
- Risk: The wrong fee applies where percentage fees meet minimum fees.
- Input that rules it out: For EUR, `Decimal("9.99")`, `Decimal("10.00")`, and `Decimal("10.01")`.
- Current behavior observed in the code: The function selects the larger of the unrounded percentage fee and the minimum, then rounds. These EUR examples all produce `0.25` after rounding, although `10.01` selects the percentage branch.
- Recommended expected contract: Establish when the minimum applies and whether comparison happens before or after rounding.
- Contract status: pending decision
- Why it matters for payments: Looking only at rounded results can conceal an incorrect branch near a pricing threshold.

#### B04 — Fractional minor units

- ID: B04
- Category: Boundary — fractional minor units
- Risk: An accepted principal cannot be represented consistently in a payable amount.
- Input that rules it out: `Decimal("0.001")`, `Decimal("1.004")`, and `Decimal("1.005")`, with `"USD"`.
- Current behavior observed in the code: Parsing and validation preserve arbitrary fractional precision. Fees and totals are rounded to two decimal places. The API returns the original parsed principal.
- Recommended expected contract: Decide whether excess precision is rejected or normalized, and perform that decision consistently before fee calculation and response generation.
- Contract status: pending decision
- Why it matters for payments: Returned principal, fee, and total can describe different precision conventions and fail to reconcile.

### Equivalence

#### E01 — Numeric representations

- ID: E01
- Category: Equivalence — numeric representations
- Risk: Equivalent amounts receive inconsistent treatment, or unintended syntax is accepted.
- Input that rules it out: `"100"`, `"100.00"`, `" 100.00 "`, `"+100.00"`, `"1e2"`, `100`, and `Decimal("100.00")`; contrast with `"1,000.00"`.
- Current behavior observed in the code: Parsing delegates to `Decimal` after string conversion and trimming. Equivalent values can retain different decimal exponents and therefore different string representations. Comma-formatted numbers are not supported.
- Recommended expected contract: Define accepted numeric syntax and distinguish numeric equivalence from canonical response formatting.
- Contract status: pending decision
- Why it matters for payments: Representation differences can complicate reconciliation, signatures, and systems that compare amounts as text.

#### E02 — Float versus exact decimal

- ID: E02
- Category: Equivalence — float versus exact decimal
- Risk: Float arithmetic changes the monetary value before parsing.
- Input that rules it out: Compare `"0.30"`, `Decimal("0.30")`, and the float expression `0.1 + 0.2`.
- Current behavior observed in the code: Floats are explicitly accepted by the annotation. Conversion through `str` preserves the float's rendered value; the expression becomes `Decimal("0.30000000000000004")`.
- Recommended expected contract: Prefer decimal strings or `Decimal` for exact amounts. If floats remain supported, define how their precision is handled.
- Contract status: pending decision
- Why it matters for payments: Small representation differences can affect thresholds or rounding in later calculations.

#### E03 — Parsed versus direct calculation inputs

- ID: E03
- Category: Equivalence — parsed versus direct calculation inputs
- Risk: The same unusable amount produces different failures depending on the entry point.
- Input that rules it out: Pass `"NaN"`, `"sNaN"`, `"Infinity"`, and `"-Infinity"` through `parse_amount`; separately pass their `Decimal` equivalents to `validate_amount`, `calculate_fee`, and `total_with_fee`.
- Current behavior observed in the code: Parsing rejects non-finite values with `AmountError`. Direct validation lacks a finiteness check: NaN comparisons can raise `InvalidOperation`; infinities reach the range comparisons. The API catches only `AmountError` and `CurrencyError`.
- Recommended expected contract: Either make every amount-consuming public function reject non-finite values consistently, or explicitly require prior parsing and enforce that boundary.
- Contract status: pending decision
- Why it matters for payments: Bypassing one helper can change a controlled rejection into an unexpected exception.

### Null/empty

#### N01 — Missing or nonnumeric amount

- ID: N01
- Category: Null/empty — missing or nonnumeric amount
- Risk: Missing or malformed amounts become valid payments or escape the intended error handling.
- Input that rules it out: `None`, `""`, `"   "`, and `"abc"` passed to `parse_amount`.
- Current behavior observed in the code: `None` raises `AmountError("amount is required")`; empty, whitespace-only, and nonnumeric strings raise `AmountError("amount must be numeric")`.
- Recommended expected contract: Reject these inputs through `AmountError`. This matches the documented purpose of the exception and the API's required amount field; exact messages are not independently specified.
- Contract status: confirmed
- Why it matters for payments: Missing input must not silently become a zero-value transaction. This path appears covered in the implementation.

#### N02 — Missing currency

- ID: N02
- Category: Null/empty — missing currency
- Risk: An unspecified currency silently selects a fee schedule.
- Input that rules it out: `None`, `""`, and `"   "` passed to `normalize_currency` or `calculate_fee` with a valid amount.
- Current behavior observed in the code: All are rejected with `CurrencyError("currency is required")`; there is no default currency.
- Recommended expected contract: Reject missing currency through `CurrencyError`, consistent with its explicit docstring and the API's required currency field.
- Contract status: confirmed
- Why it matters for payments: A currency determines the units and fee schedule. This path appears covered in the implementation.

#### N03 — Direct helper calls

- ID: N03
- Category: Null/empty — direct helper calls
- Risk: Callers assume calculation helpers provide the same input handling as parsing.
- Input that rules it out: `None` or `""` passed directly as the amount to `validate_amount`, `round_money`, `calculate_fee`, and `total_with_fee`.
- Current behavior observed in the code: These helpers expect `Decimal` and do not normalize missing inputs. Depending on the helper, comparison or method access raises `TypeError` or `AttributeError`.
- Recommended expected contract: Document these as functions requiring validated `Decimal` values, or provide consistent domain errors at their public boundaries.
- Contract status: pending decision
- Why it matters for payments: Integration code needs to know which entry point guarantees controlled input rejection. These examples are outside the declared parameter types, so they are not automatically defects.

### Business contract

#### C01 — Fee schedule and formula

- ID: C01
- Category: Business contract — fee schedule and formula
- Risk: Plausible-looking constants implement the wrong pricing agreement.
- Input that rules it out: `Decimal("1.00")` and `Decimal("100.00")` for each supported currency, compared with an approved pricing schedule.
- Current behavior observed in the code: Rates are USD `2.9%`, EUR `2.5%`, and COP `1.9%`; minimums are `0.30`, `0.25`, and `900.00`. The minimum is a floor, rather than a fixed charge added to the percentage.
- Recommended expected contract: Confirm the rates, units, minimums, formula, and applicability with an authoritative pricing source.
- Contract status: pending decision
- Why it matters for payments: A floor and an additive charge produce materially different fees. The constants alone do not establish the intended agreement.

#### C02 — Rounding rule and sequence

- ID: C02
- Category: Business contract — rounding rule and sequence
- Risk: Tie handling or intermediate rounding changes the final charge.
- Input that rules it out: `round_money(Decimal("1.005"))`, `round_money(Decimal("1.015"))`, and `total_with_fee(Decimal("10.005"), "EUR")`.
- Current behavior observed in the code: Rounding uses half-even to two places: the first two examples yield `1.00` and `1.02`. Fees are rounded before addition; totals are then rounded again. The EUR total example yields `10.26`.
- Recommended expected contract: Specify the rounding mode and the exact stages at which principal, fees, and totals are rounded.
- Contract status: pending decision
- Why it matters for payments: Different rounding sequences can produce cent-level differences across charging and reconciliation systems.

#### C03 — Currency support and monetary units

- ID: C03
- Category: Business contract — currency support and monetary units
- Risk: A universal scale or incorrectly configured currency produces unusable amounts.
- Input that rules it out: `"usd"`, `" USD "`, `"EUR"`, `"COP"`, and `"GBP"`; fractional COP amounts such as `Decimal("1000.005")`.
- Current behavior observed in the code: Currency strings are trimmed and uppercased. Membership in `FEE_RATES` defines support; other codes raise `CurrencyError`. Every currency uses `CENT = Decimal("0.01")`, and calculation assumes a matching minimum-fee entry.
- Recommended expected contract: Explicitly define supported currencies, major versus minor units, permitted precision, and complete fee configuration for each currency.
- Contract status: pending decision
- Why it matters for payments: Unit or scale mismatches can misstate a charge substantially. A missing minimum-fee entry would also cause an unhandled `KeyError`.

#### C04 — Deterministic decimal arithmetic

- ID: C04
- Category: Business contract — deterministic decimal arithmetic
- Risk: Ambient decimal settings or extreme precision alter results or trigger unexpected exceptions.
- Input that rules it out: Calculate the same ordinary payment under different decimal precisions; examine `Decimal("1E-1000")` and `Decimal("1." + "0" * 100 + "1")`.
- Current behavior observed in the code: Multiplication, addition, and quantization use the active decimal context. There is no local context, input precision limit, or translation of calculation-time decimal exceptions into domain errors.
- Recommended expected contract: Define supported precision and magnitude, use sufficient controlled arithmetic precision, and reject unsupported representations predictably.
- Contract status: pending decision
- Why it matters for payments: The same accepted amount should have a stable charge across execution environments; unexpected arithmetic exceptions can interrupt payment processing.
