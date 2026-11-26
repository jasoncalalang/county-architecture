# ADR 0015: Added lock targets follow the canonical four

Date: 2026-11-26
Status: accepted

Supersedes the lock-order bullet of
[ADR 0013](0013-ordering-persistence-uses-panache.md). The rest of ADR 0013
stands.

## Context

ADR 0013 fixes one pessimistic lock order: reseller, business, products by
ascending id, order. "A new workflow that takes more than one lock keeps
this order." That list covers v1. The Beyond-v1 build that
[ADR 0014](0014-begin-beyond-v1-build.md) accepts adds rows the list does
not name:

- assigning a reseller's price tier share-locks the `price_tier` row after
  the reseller, so a concurrent deactivation commits first
  ([build plan §2.8](../design/2026-11-23-beyond-v1-itemcount-build-plan.md));
- accepting a quote locks the quote after the business, then inserts the
  order it produces (§3.8);
- issuing or settling a payment link locks the `payment_link` row after the
  order (§6.8).

Amending ADR 0013 with each position would need a new ADR per feature and
would grow a list nobody checks against. A rule that every addition must
satisfy keeps the ADR short and gives reviewers one test to apply.

Alternatives considered: a longer canonical list (one ADR per feature, and
the list must be totally ordered in advance); advisory locks for the new
rows (a second locking mechanism, which ADR 0013 keeps for first sign-in
only); optimistic checks with retries for the new rows (ADR 0013 already
weighed and deferred retries).

## Decision

- Reseller, business, products by ascending id and order keep the order ADR
  0013 gives them. No workflow reorders them or takes one out of turn.
- A workflow may lock a row outside those four only after every one of the
  four it locks. No workflow locks such a row before one of the four.
- The rule is about rows that already exist. A row a workflow inserts in its
  own transaction is not a lock target, because no other transaction can
  wait on it before commit.
- The added lock targets are listed, in order, in the lock note of
  `api/README.md` in the county repository. A workflow that takes more than
  one added target takes them in that order, and several rows of one target
  in ascending id. The list changes in the same pull request that adds a
  target.
- A new target that cannot fit the rule needs its own ADR.

## Consequences

- Every lock a workflow takes now sits in one total order: the canonical
  four, then the README list. Two workflows that follow it cannot deadlock
  on these rows.
- The first entry is `price_tier`, from per-reseller price tiers. Quotes and
  payment links add their lines when they are built, with no further ADR.
- Reviewers check a new workflow's locks against ADR 0013, this rule and
  the README list. Tests that wait on a real lock, as ADR 0013 requires,
  cover each added target.
- The list lives in the code repository, so it can drift from the code.
  Keeping it in the same pull request as the lock is the guard.
