# Reference enums

## Language — `FR | EN`

Portal language, notification templates, statement language. Referenced by the portal,
notifications, and reporting.

## BusinessDayConvention

How a date landing on a non-business day is rolled. Used by settlement and coupon/maturity
dating via the Calendar service.
Proposed set: `FOLLOWING` · `MODIFIED_FOLLOWING` · `PRECEDING`.

## DayCountConvention

The accrual day-count basis — a hard dependency of the bond / BRH / deposit / fee-accrual
calculations, so the exact members matter.
Proposed set: `ACTUAL_365` · `ACTUAL_360` · `ACTUAL_ACTUAL` · `THIRTY_360` · `THIRTY_E_360`
(the requirements already call for at least `ACTUAL_365` and `THIRTY_360`; add others only as an
instrument needs them).

## PeriodFrequency

Coupon / contribution / fee period. Referenced by instruments (coupons), pension
(contributions), and fees (periodic charges).
Proposed set: `ANNUAL` · `SEMI_ANNUAL` · `QUARTERLY` · `MONTHLY` · `WEEKLY` · `DAILY` ·
`AT_MATURITY` · `NONE`.
