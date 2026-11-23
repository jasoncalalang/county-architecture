# Beyond-v1 build plan informed by Itemcount

Date: 2026-11-23
Status: proposed

## 1. Scope reality

county v1 is a deliberate thinnest-viable-slice. The
[v1 design](2026-10-16-v1-design.md) states it plainly: "Version one is a
demo, not a pilot," and its *Out of scope for v1* list names "discrepancy
reports, payments beyond a paid flag, ... price tiers and minimum order
quantities" explicitly. The
[functional spec](2026-10-31-v1-functional-spec.md) repeats the line in §2
("Not in v1, by decision: ... payments beyond the flag, price tiers,
minimum order quantities, ... editing an order after placement (ADR 0012)")
and lists ten deferred features in §12 *Beyond v1*. [ADR
0012](../adr/0012-orders-are-immutable-after-placement.md) defers discrepancy
reports by name and anticipates their return.

This document proposes how to pull five of those deferred features in, after
v1, faithfully derived from the Itemcount reverse-engineering report
(`itemcount-deep-rea.md`, §3 per-domain, §4 ER/capability matrix, §5.2 "What
county should build next"). Itemcount is the reference, not the target:
county keeps its REST surface (no GraphQL — a hard owner preference), its
Flyway-owned schema with `(tenant_id, id)` composite uniqueness and
tenant-scoped composite FKs, its canonical pessimistic lock order, its typed
view/input DTOs, and immutability after placement. Each feature below is a
minimal, faithful analogue of an Itemcount capability — never a port of
Itemcount's full machinery.

The five features, in the order §5.2 of the report ranks them and the order
this document recommends building them:

| # | Feature | §5.2 rank | Effort | New tables | Touches placement lock path |
|---|---|---|---|---|---|
| 1 | Per-reseller price tiers | 1 (highest value) | M | `price_tier`, `product_tier_price` | yes (no new placement lock) |
| 2 | Quotes → order | 3 | L | `quote`, `quote_line` | yes (quote in order's slot) |
| 3 | Receive-with-discrepancy | 2 (highest correctness) | M | `discrepancy_report`, `discrepancy_line` | no new root |
| 4 | MOQ + order-create windows | 4 | M | none (additive columns) | no new lock targets |
| 5 | Online payment link | 5 (lowest leverage) | M | `payment_link`, `payment_event` | tail child after order |

### 1.1 Conventions every section honours

These are the invariants from [ADR 0009](../adr/0009-flyway-migrates-at-api-start.md),
[ADR 0012](../adr/0012-orders-are-immutable-after-placement.md) and
[ADR 0013](../adr/0013-ordering-persistence-uses-panache.md) that constrain
every proposal and are not restated in each section:

- **Schema.** Flyway owns it; Hibernate never generates or alters it (ADR
  0009/0013). New migrations continue from `V2__owner_identities.sql` as
  `V3__*.sql`. New root tables carry identity PK + `UNIQUE (tenant_id, id)`;
  every table carries tenant-scoped composite FKs, mirroring
  `V1__ordering.sql`. Child tables that nothing references (`order_line` in
  V1; the price, line and event tables here) do not need the composite
  unique key. Migrations are forward-only and expand-first (ADR 0009 rules
  1 and 2): new columns are nullable or defaulted so the schema works for
  the previous release.
- **Tenant scope** comes only from the Google subject (owner path, via
  `Owners.tenant(subject).id`) or the reseller token (reseller path, via
  `Reseller.byToken → reseller.tenantId`), never from a request field (ADR
  0013). A resource in another tenant is a 404 (ADR 0013); an unknown or
  blocked token is a 404 too (ADR 0003).
- **Lock order** is always reseller → business → products by ascending id →
  order; any new multi-lock workflow fits this order (ADR 0013).
- **Immutability.** An order's lines, note, delivery date and total never
  change after placement; only status and paid change (ADR 0012). Captured
  `order_line` prices never change.
- **Wire contract.** Explicit typed `Views` records out, `Inputs` records
  in; entities are never serialised. Errors are `Problem(status, detail)`
  via the `Problems` mapper (SQLState 23505 → 409). The committed OpenAPI
  contract is checked for drift, so every new DTO must be registered and
  every change to an existing one is scrutinised.

---

## 2. Feature 1 — Per-reseller price tiers

A wholesaler assigns each reseller a named price tier; a tier carries
explicit per-product overrides; the reseller's catalog and placed-order
lines use the tier-resolved effective price, falling back to the flat
`product.price`.

### 2.1 Itemcount evidence

Report §3.3 ("B2B Ordering, Partners & Consignments"), *Pricing* line: "two-
tier (`PricingType` WHOLESALE/RETAIL) + `ItemPricingOption` + per-supplier
bulk pricing + per-sub-account templates/RevenueOps custom pricing;
visibility gated by `revealPrices`/`hideGroupPrice`/`AllowWholesalePricing`."
The central `Item` (§3.2) carries `price`, `cost`, `groupPrice`,
`wholesalePrice`, `wholesaleMinQty`. §5.2 ranks this **#1**: "Price tiers /
per-reseller pricing — county has one flat `Product.price`; Itemcount's
`ItemPricingOption` + WHOLESALE/RETAIL tier is the single highest-value add
for a wholesaler→reseller tool, and slots directly onto the existing catalog
+ order-capture." The §4 comparison scores it Itemcount full / county none.

county's minimal slice intentionally drops Itemcount's per-sub-account
RevenueOps/custom-pricing machinery and its visibility gates. The faithful
minimal analogue is a **named tier** (= `PricingType`) + **per-item
override** (= `ItemPricingOption`) assigned per reseller.

### 2.2 county current state

One flat column: `product.price NUMERIC(12,2) CHECK (price>0)` in
`V1__ordering.sql`. `Owners.product()` sets it. It is read in two places:
`Orders.catalog()` maps each active product to
`Views.CatalogProduct(p.id, p.name, p.unit, p.price)`; `Orders.place()`
reads `product.price` under `PESSIMISTIC_READ`, computes
`Money.line(product.price, qty)`, and snapshots it into `order_line.price` /
`order_line.total`. The reseller row has no pricing attribute. There is no
price-list entity; every reseller of a business sees the same catalog price.

### 2.3 Scope status — deferred past v1 by the functional spec (doubly)

functional-spec §12 *Beyond v1*: "| Price tiers, per-reseller price lists,
minimum order quantities | Every reseller pays the same in the reference
customer. |". Also §2 *Not in v1, by decision*: "... price tiers, minimum
order quantities ...". No ADR specifically rejects it; ADR 0012 governs only
immutability, which this design respects.

### 2.4 Flyway model

This section and the rest of §2 describe the reference implementation in
[county PR #4](https://github.com/jasoncalalang/county/pull/4), which built
this feature tests-first. Where the first draft of this document differed
from what was built, §2.8 says why.

`V3__price_tiers.sql`, mirroring V1 style. `product.price` stays as the
base/fallback (the implicit "standard" tier); no existing column changes.

```sql
CREATE TABLE price_tier (
  id        BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id BIGINT NOT NULL REFERENCES business(id),
  name      VARCHAR(80) NOT NULL,
  active    BOOLEAN NOT NULL DEFAULT TRUE,
  UNIQUE (tenant_id, id)
);
CREATE UNIQUE INDEX unique_active_price_tier
  ON price_tier (tenant_id, lower(name)) WHERE active;

ALTER TABLE reseller ADD COLUMN price_tier_id BIGINT;  -- NULL = base product.price
ALTER TABLE reseller ADD CONSTRAINT reseller_price_tier_fk
  FOREIGN KEY (tenant_id, price_tier_id) REFERENCES price_tier(tenant_id, id);

CREATE TABLE product_tier_price (
  id            BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id     BIGINT NOT NULL,
  product_id    BIGINT NOT NULL,
  price_tier_id BIGINT NOT NULL,
  price         NUMERIC(12,2) NOT NULL CHECK (price > 0),
  UNIQUE (tenant_id, product_id, price_tier_id),   -- at most one price per (product, tier)
  FOREIGN KEY (tenant_id, product_id)    REFERENCES product(tenant_id, id),
  FOREIGN KEY (tenant_id, price_tier_id) REFERENCES price_tier(tenant_id, id)
);
CREATE INDEX product_tier_price_by_tier
  ON product_tier_price (tenant_id, price_tier_id);
```

The price table is named for its owner. A tier price belongs to a product
and is guarded by that product's lock (§2.8), so it is `product_tier_price`
and its endpoints hang off `/products/{id}`.

New Panache entities `PriceTier` and `ProductTierPrice` (`PanacheEntityBase`,
`@Column` mappings, a static `PriceTier.inTenant(tenant, id, lock)` finder
like `Product`/`Reseller`). `Reseller` gains a nullable `priceTierId`.
Effective price for `(reseller, product)` = `product_tier_price.price` when
`reseller.price_tier_id` is set and a row exists for that product, else
`product.price`. `order_line` still stores the one resolved price per line,
so the catalog and order contracts keep their shape.

### 2.5 REST endpoints

| Method/path | Caller | Effect |
|---|---|---|
| `GET /price-tiers` | owner | list this business's tiers, active first → `List<Views.PriceTier>` |
| `POST /price-tiers` | owner | create a named tier, 201 → `Views.PriceTier` (`Inputs.PriceTier{name, active?}`) |
| `PUT /price-tiers/{id}` | owner | rename / activate / deactivate → `Views.PriceTier` |
| `GET /price-tiers/{id}/prices` | owner | the tier with every product's base price and optional tier price → `Views.TierPrices` |
| `PUT /products/{id}/tier-prices/{tierId}` | owner | set the product's price under a tier, `Inputs.TierPrice{price}` → `Views.TierPrices` |
| `DELETE /products/{id}/tier-prices/{tierId}` | owner | clear it, 204; the product falls back to its base price |
| `PUT /resellers/{id}/price-tier` | owner | assign or clear a reseller's tier, `Inputs.ResellerTier{priceTierId?}` → `Views.Reseller`; 404 if not this tenant's tier, 422 if the tier is inactive |
| `GET /order/{token}` | reseller (existing) | **no new endpoint**; `Orders.catalog()` now returns tier-resolved prices in the existing `Views.CatalogProduct.price` |
| `POST /order/{token}/orders` | reseller (existing) | `Orders.place()` now snapshots the tier-resolved price; no signature change |

Assignment is its own endpoint rather than a field on `PUT /resellers/{id}`.
That request replaces name and mobile; a client that predates tiers and
omits the new field would silently clear the tier on every edit.

The `DELETE` is the first delete in the codebase. Justify it as config:
tier-price rows carry no order history and are never referenced by placed
orders, so clearing one is safe. The v1 "nothing is deleted" rule
(functional-spec §5.7) is itself a v1-scoped rule about retiring *history*
(deactivate/block/cancel); a pricing-config row has no history to keep. If
the owner prefers to preserve the ethos exactly, switch to a null-price
sentinel `PUT` instead. The reference implementation uses `DELETE`; the
draft ADR lists it for sign-off.

### 2.6 Views / Inputs

New: `Views.PriceTier(long id, String name, boolean active)`;
`Views.TierPrices(long id, String name, boolean active,
List<Views.TierProductPrice> products)` for the owner tier-editor screen,
where `Views.TierProductPrice(long productId, String productName, @enum
unit, BigDecimal basePrice, @nullable BigDecimal tierPrice)` lists every
product of the business, active first; `Inputs.PriceTier(@NotBlank
@Size(max=80) String name, Boolean active)`; `Inputs.TierPrice(@NotNull
@DecimalMin(value="0", inclusive=false) @Digits(integer=10, fraction=2)
BigDecimal price)` — same validation as `Inputs.Product.price`;
`Inputs.ResellerTier(@Positive Long priceTierId)`, where null clears the
assignment.

Changed, kept backward-compatible: `Views.Reseller` gains a nullable
`priceTierId` as its last field, so the resellers screen can show the
assignment; the dashboard resolves the name from `GET /price-tiers`.
`Inputs.Reseller` is unchanged. **Unchanged:** `Views.CatalogProduct` and
`Views.OrderLine` (the `price` field now simply carries the resolved value),
so the committed OpenAPI contract and the reseller order page stay stable, and
"seeing prices other than your own current price" stays out of scope.

### 2.7 Tenant scoping

Unchanged boundary. All new owner queries filter on `tenantId` and use the
`(tenant_id, *)` composite FKs. On the reseller side the applicable tier is
`reseller.price_tier_id` loaded from the token-resolved reseller row — the
client never sends a tier id, so a reseller cannot select a cheaper tier. On
the owner side, `Inputs.ResellerTier.priceTierId` and the ids in the tier
and tier-price paths are intra-tenant references the owner chooses; they are
constrained to the owner's own tenant by the composite FK **and** by looking
each one up with `PriceTier.inTenant` / `Product.inTenant` /
`Reseller.inTenant` → 404 otherwise. This preserves ADR 0013: the tier id
sets an owned association, it does not widen the tenant boundary.

### 2.8 Lock order and immutability (ADR 0012/0013)

**No new lock on the placement path.** A product's tier prices are guarded by
that product's lock, the one ADR 0013 already defines:

- `Orders.place()` is unchanged up to the product loop: reseller and
  business `PESSIMISTIC_WRITE`, then each product `PESSIMISTIC_READ` in
  ascending id. Immediately after taking a product's lock it reads that
  product's tier price for `reseller.priceTierId`, if any, and resolves
  `effectivePrice = tierPrice ?? product.price` before `Money.line(...)`.
- Setting or clearing a tier price takes the product `PESSIMISTIC_WRITE`
  first, exactly as a base-price edit does, then writes the
  `product_tier_price` row. A placement that reaches the same product waits
  and then reads the committed value.
- Assigning a reseller's tier takes the reseller `PESSIMISTIC_WRITE`, the
  first lock placement takes, so a placement sees the assignment either
  wholly before or wholly after. It then takes the `price_tier`
  `PESSIMISTIC_READ` (reseller → tier), so a deactivation running at the
  same time, which holds the tier's write lock, finishes first and the 422
  active check sees its result.
- Renaming or (de)activating a tier locks only the `price_tier` row;
  placement never reads that row.

Placement takes no lock beyond the ADR 0013 chain, so the order reseller →
business → products → order stands as written there. Tier assignment is the
one workflow that adds a lock target: the `price_tier` row, taken after the
reseller. That is a reseller → price_tier pair outside placement, which
ADR 0013's order does not list. No cycle is possible, because deactivating
or renaming a tier takes only the tier lock and placement never takes it, so
nothing holds the tier lock while waiting for a reseller. The planned ADR
0013 amendment named in draft ADR 0014 records the pair alongside the quote and `payment_link`
positions.

*Alternative considered:* the first draft of this document gave the tier
its own lock slot between business and products (placement share-locks the
`price_tier` row and reads all its prices once; a tier-price edit
write-locks the tier). It reads every price in one query, but it extends
the lock chain on the hottest path, needs an ADR 0013 amendment, and makes
every tier-price edit block every placement by a reseller on that tier
rather than only placements containing that product. The product-lock
guard was built instead. Its cost is one indexed tier-price lookup per
order line for resellers on a tier.

Fully compatible with ADR 0012. The resolved effective price is snapshotted
into `order_line.price` / `order_line.total` at placement, exactly as
`product.price` is today; nothing new becomes mutable. Editing a tier
price, clearing one, or reassigning a reseller's tier affects only orders
placed afterward. This adds one row to functional-spec §5.7 *What changes
affect what*:

| Change | Existing orders | New orders |
|---|---|---|
| Tier price edited / reseller's tier changed | unchanged | new effective price |

matching the existing "Price edited | unchanged | new price" row.

**Tier retirement — open decision.** Deactivating a tier stops new
assignments (422) and frees its name for reuse. Resellers already assigned
keep the tier's prices until the owner reassigns them. That is what the
reference implementation does, pinned by a test, because it needs no lock
on the tier at placement. The alternatives are to refuse deactivation while
resellers are assigned (409) or to charge the base price while a tier is
inactive; either makes placement or deactivation coordinate on the
`price_tier` row. The owner picks one before this merges.

### 2.9 Tests first

Built in `PriceTierApiTest`, `PriceTierIsolationTest` and
`PriceTierConcurrencyTest`:

- **API shapes:** `POST /price-tiers` 201 with `Views.PriceTier`; duplicate
  active tier name → 409 (`unique_active_price_tier`) on create and on
  rename; `PUT` rename/deactivate; set and clear a tier price; bad price
  (≤0, >2dp) and blank name → 422; non-positive `priceTierId` → 422.
- **Effective-price resolution:** reseller with a tier and a tier price →
  that price in catalog and `order_line`; tier but no price for a product →
  base `product.price`; no tier → base price everywhere.
- **Tenant isolation:** another tenant's tier, product or reseller is a 404
  on every new path, including the delete; the tier list shows only this
  tenant's tiers; every new endpoint requires authentication.
- **Immutability (ADR 0012):** place an order, edit the tier price;
  existing `order_line.price` and `order.total` unchanged; only subsequent
  placements see the new price.
- **Concurrency (ADR 0013):** a placement racing an in-flight tier-price
  edit waits on the product lock and captures the committed tier price.
  An assignment racing a tier deactivation waits on the tier lock and then
  rejects the now-inactive tier with 422
  (`assignmentWaitsForTierDeactivationAndThenRejectsTheInactiveTier`).
- **Tier retirement:** an inactive tier cannot be assigned (422); a
  reseller already on it keeps its prices (see the open decision in §2.8).
- **OpenAPI drift:** `PriceTierView`, `TierPricesView`,
  `TierProductPriceView` and the new paths are in the committed contract.

### 2.10 Effort and recommendation

Effort **M**. Build it first out of the v1 gate. It is explicitly deferred
by functional-spec §12, but it is §5.2's #1 build-next and the highest-
leverage post-v1 increment, slotting onto the existing catalog + placement
with no new infrastructure. Build after a pilot wholesaler actually needs
differentiated pricing (the reference customer charges every reseller the
same), but make it the first feature, not a v1 addition.

**Risks.** (1) The owner-chosen `priceTierId` on
`PUT /resellers/{id}/price-tier` is a request field — validate it with
`PriceTier.inTenant` **and** the composite FK, or it reads as
request-derived scope; the reviewer must confirm the lookup, not just the
FK. (2) The tier-price read in `place()` must stay after that product's
lock is taken; reading all tier prices up front would look like a harmless
optimisation and would let a placement capture a price that an in-flight
edit is replacing. The concurrency test guards this. (3) The `DELETE` is
the first delete — justify as config or use a null-price sentinel.
(4) Resist scope creep toward RevenueOps per-sub-account/custom pricing and
visibility gates. (5) A deactivated product leaves its tier-price rows in
place; they are harmless, because the catalog lists only active products
and placement rejects inactive ones. (6) No dashboard screen exists for
tiers yet; the UI design needs a tier editor and a tier picker on the
resellers screen before this is usable by an owner.

---

## 3. Feature 2 — Quotes → order

Wholesaler-authored, price-capturing quotes that a reseller accepts (converts
to an immutable `PurchaseOrder`) or declines on their existing public token
link.

### 3.1 Itemcount evidence

Report §3.3 (OBS): public unauthenticated document routes include
`/sales/quotes/[id]` with a "password-gated accept"; the workflow is
quote→invoice via `ConvertQuoteToInvoice` with gross-profit-margin per line;
realtime inventory lists "quote accept/decline" as a domain event; the ER
sketch shows `Quote (→ Invoice / SO / PO / WO)` off the Transaction core. The
§4 matrix scores "Quotes (→invoice/SO/PO/WO, accept/decline)" Itemcount ● /
county ○. §5.2 ranks it **#3**: "`CreateQuote` + accept/decline before an
order is placed; reuses the existing order capture and public-document
pattern county already has for invites."

county's slice drops Itemcount's Transaction-is-everything model, GPM, and
the SO/PO/WO fan-out: a county quote converts only to a sell-side
`PurchaseOrder`.

### 3.2 county current state

No quote concept exists. Directly reusable: the public token document pattern
in `ResellerResource @Path("/order/{token}")` (GET catalog, POST orders
idempotent 200 replay / 201 create, GET/cancel); `Orders.place()` with its
captured-snapshot `OrderLine`, per-business numbering via
`business.next_order_number`, and idempotency via
`purchase_order.idempotency_key` + `UNIQUE (tenant_id, idempotency_key)`; the
canonical lock order; the owner side in `OwnerResource @Authenticated`; the
`OrderRules` state machine; the typed-DTO discipline; REST only.

### 3.3 Scope status — excluded from v1 by omission; proposed post-v1 by the report, not the spec

Honest reading: functional-spec §2 declares the v1 feature list exhaustive
("Everything in v1, in one table"), and no row mentions quotes. The §2 *Not
in v1* sentence does not name quotes, and §12 *Beyond v1* lists ten features,
none of them quotes. So the spec neither includes nor explicitly defers
quotes — they are excluded by omission from an exhaustive list. The only
document that places the feature post-v1 is the RE report §5.2 item 3. No ADR
addresses quotes. This is the weakest scope mandate of the five and most
needs owner sign-off before build.

### 3.4 Proposed Flyway model

`V3__quotes.sql`, matching V1 style.

```sql
ALTER TABLE business ADD COLUMN next_quote_number BIGINT NOT NULL DEFAULT 1;

CREATE TABLE quote (
  id            BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id     BIGINT NOT NULL REFERENCES business(id),
  reseller_id   BIGINT NOT NULL,
  number        BIGINT NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL,
  expires_on    DATE NOT NULL,
  delivery_date DATE NOT NULL,
  status        VARCHAR(9) NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft','sent','accepted','declined','expired','cancelled')),
  total         NUMERIC(12,2) NOT NULL,
  note          VARCHAR(500) NOT NULL DEFAULT '',
  accepted_order_id BIGINT,
  sent_at TIMESTAMPTZ, accepted_at TIMESTAMPTZ,
  declined_at TIMESTAMPTZ, cancelled_at TIMESTAMPTZ,
  FOREIGN KEY (tenant_id, reseller_id)       REFERENCES reseller(tenant_id, id),
  FOREIGN KEY (tenant_id, accepted_order_id) REFERENCES purchase_order(tenant_id, id),
  UNIQUE (tenant_id, number), UNIQUE (tenant_id, id)
);
CREATE TABLE quote_line (
  id           BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id    BIGINT NOT NULL,
  quote_id     BIGINT NOT NULL,
  product_id   BIGINT NOT NULL,
  product_name VARCHAR(80) NOT NULL,
  unit         VARCHAR(4) NOT NULL,
  quantity     INTEGER NOT NULL CHECK (quantity BETWEEN 1 AND 9999),
  price        NUMERIC(12,2) NOT NULL,
  total        NUMERIC(12,2) NOT NULL,
  FOREIGN KEY (tenant_id, quote_id)   REFERENCES quote(tenant_id, id),
  FOREIGN KEY (tenant_id, product_id) REFERENCES product(tenant_id, id),
  UNIQUE (quote_id, product_id)
);
CREATE INDEX quote_reseller ON quote (tenant_id, reseller_id, created_at DESC);
CREATE INDEX quote_open     ON quote (tenant_id, status);
```

Panache entities `Quote`/`QuoteLine` mirror `PurchaseOrder`/`OrderLine`, each
with an `inTenant(tenant, id, resellerOrNull, lock)` finder. Quote prices are
a captured snapshot like `OrderLine` — the quote is the one legitimate way a
wholesaler offers a price that differs from the live flat `product.price`,
and that captured price is copied verbatim into `order_line` on accept. No
idempotency column on `quote`: accept reuses `purchase_order.idempotency_key`
(the accept body carries the UUID, the created order stores it), so replay
flows through the existing `UNIQUE (tenant_id, idempotency_key)` path and an
already-accepted quote replays via `accepted_order_id`.

### 3.5 REST endpoints

Owner (`OwnerResource`, `@Authenticated`):

- `GET /quotes?status=&resellerId=` → `Views.QuoteSummary`
- `POST /quotes` — `Inputs.Quote{resellerId, lines[], note, expiresOn}`;
  201 `Views.Quote` in `draft`
- `GET /quotes/{id}` → `Views.Quote`
- `POST /quotes/{id}/send` — draft→sent, makes it visible on the link
- `POST /quotes/{id}/cancel` — draft|sent→cancelled

Reseller token (`ResellerResource`, public):

- `GET /order/{token}/quotes` — this reseller's sent/open quotes
- `GET /order/{token}/quotes/{id}` — the public quote document; 404 if not
  this reseller's or not visible (token 404 posture)
- `POST /order/{token}/quotes/{id}/accept` — `Inputs.AcceptQuote{idempotencyKey}`;
  returns `Views.Order`, 201 on create / 200 on idempotent replay, exactly
  like `POST /order/{token}/orders`
- `POST /order/{token}/quotes/{id}/decline` — sent→declined

### 3.6 Views / Inputs

New only; existing contracts untouched. `Views.Quote(id, resellerId, number,
createdAt, deliveryDate, expiresOn, status[enum], total, note,
acceptedOrderId?, sentAt/acceptedAt/declinedAt/cancelledAt?, List<QuoteLine>
lines)`; `Views.QuoteLine(productId, productName, unit, quantity, price,
total)` — structurally identical to the existing order-line view;
`Views.QuoteSummary(id, number, resellerId, createdAt, deliveryDate,
expiresOn, status, total)`. `Inputs.Quote(@NotNull @Positive Long resellerId,
@NotEmpty @Size(max=50) List<@Valid QuoteLine> lines, @Size(max=500) String
note, @NotNull @FutureOrPresent LocalDate expiresOn)`; `Inputs.QuoteLine`
same shape as the existing `Inputs.Line`; `Inputs.AcceptQuote(@NotNull UUID
idempotencyKey)`. The accept endpoint returns the existing `Views.Order`
unchanged (built by the same private `Orders.view`), so the order wire
contract is stable.

`Inputs.Quote` carries no delivery date, on purpose: the server computes it,
as it does for every order (functional-spec §5.1). `quote.delivery_date` is
the date `DeliveryDates.compute` gives when the quote is created, shown to
the reseller as an estimate. **Open point:** whether the accepted order
keeps that date or recomputes it at accept, as `place()` would. Recomputing
is the consistent choice, because a quote accepted after a cutoff would
otherwise promise a date the kitchen can no longer meet; decide it before
the quote slice starts and test it.

### 3.7 Tenant scoping

Owner endpoints resolve tenant via `Owners.tenant(subject).id`; the
`resellerId` in `Inputs.Quote` is not a tenant selector — it is looked up with
`Reseller.inTenant(ownerTenant, resellerId)` and 404s otherwise. Reseller
endpoints resolve tenant **and** reseller from `Reseller.byToken(token)`
(which excludes blocked tokens); a quote fetched/accepted on a token is
additionally constrained by `reseller_id` = the token's reseller, so reseller
Y cannot see or accept a quote cut for reseller X (404). No request body ever
sets `tenant_id`, `reseller_id`, `price`, `number`, or `status`.

### 3.8 Lock order and immutability (ADR 0012/0013)

Fits the canonical chain. **CreateQuote (owner):** business
`PESSIMISTIC_WRITE` to allocate `next_quote_number` gap-free (same as
`place()` for `next_order_number`), then products `PESSIMISTIC_READ`
ascending id to snapshot name/unit/price, then insert quote+lines.
**Accept (reseller):** reseller `PESSIMISTIC_WRITE` → business
`PESSIMISTIC_WRITE` (order numbering + idempotency) → quote
`PESSIMISTIC_WRITE` → new order; conversion copies captured quote prices and
does **not** re-read `product.price`, so products are skipped and the chain
reseller→business→quote→order is a safe prefix with the quote in the order's
slot — no new deadlock class. New lock subject: the quote row, taken after
business and immediately before the order it produces; send/cancel/decline
lock only that single row.

Consistent with ADR 0012 and extends its spirit. A quote is mutable only
while draft/sent via whole-row state transitions; there is no quote line-edit
endpoint — a wrong quote is cancelled and re-created, exactly as ADR 0012
handles orders. Captured `quote_line` prices never change once written.
Acceptance creates a brand-new `PurchaseOrder` whose `order_lines` copy the
quote's captured prices; from that moment it is immutable per ADR 0012. A
quote is therefore the one legitimate mechanism to place an order at a price
other than the live `product.price`, and ADR 0012 freezes it.

**Deliberate divergence from the live `/order` path, to document or it reads
as a bug:** `Orders.place()` reads live `product.price` and rejects
deactivated products (functional-spec §5.7); conversion instead honours the
quote's frozen snapshot, so a later price-list edit or deactivation does not
alter an accepted quote or its order.

**Not a divergence:** accept marks the reseller as having ordered
(`hasOrdered = true`, status `active`), exactly as `place()` does. That
write is why accept takes the reseller lock for writing.

### 3.9 Tests first

- **Happy path:** owner `POST /quotes` 201 snapshotting current price/name/
  unit; `send` flips to sent; reseller GETs the document; `accept` returns
  201 `Views.Order` whose lines equal the quote lines; order visible via
  owner `GET /orders/{id}`.
- **Idempotent accept:** two accepts with the same key create exactly one
  order, second returns 200 replay (same number).
- **Already-terminal accept:** accepting an accepted quote returns its
  `accepted_order_id` order (no second order); accepting a declined/expired/
  cancelled quote → 409 naming the status.
- **Tenant isolation (owner):** subject of tenant B → 404 on tenant A's
  quote.
- **Tenant/reseller isolation (token):** a quote cut for reseller X → 404 on
  reseller Y's token GET and accept; blocked token → 404 everywhere.
- **Scope-not-from-field:** `Inputs.Quote.resellerId` pointing at another
  tenant's reseller → 404; no body sets tenant/price/number/status.
- **Concurrency:** two concurrent accepts (distinct keys) → exactly one
  order; the loser replays or 409s, never a duplicate.
- **Immutability vs price list:** edit/deactivate the product after send,
  then accept — `quote_line.price` and resulting `order_line.price` are the
  captured price, and a deactivated product does not block conversion.
- **`QuoteRules` domain unit tests:** legal set {draft→sent, draft→cancelled,
  sent→accepted(reseller), sent→declined(reseller), sent→cancelled(owner),
  sent→expired}; every other transition throws; owner cannot accept, reseller
  cannot send/cancel.
- **Expiry:** a quote past `expires_on` is treated as expired on read and
  refuses accept (409).
- **Numbering:** quote numbers per-business, gap-free, allocated in the
  transaction; cancelled quotes keep their number (parity with
  functional-spec §5.2).
- **OpenAPI drift:** new quote `Views`/`Inputs` registered; existing
  order views reused unchanged.

### 3.10 Effort and recommendation

Effort **L**. Build second, after price tiers. Quotes depend on a clean
"price that isn't the flat `product.price`" primitive; introducing tiers
first lets the quote line reuse that captured-price model instead of hard-
coding a second pricing path. Quotes remain an excellent #2 because they
reuse the token document + idempotent capture wholesale and need no new
infrastructure.

**Risks.** Pricing overlap with tiers — model the captured-snapshot price as
the shared primitive. The divergent conversion semantics (honour quote
snapshot, ignore deactivation) must be documented. Expiry needs a lazy-on-
read check, not a background job: the API has no scheduler today and this
feature should not introduce one. The
token=credential choice drops Itemcount's password gate — acceptable under
ADR 0003, but anyone with the link can accept; keep the block/reissue
mitigation. Quote and order numbers are separate sequences (`#Q7` vs `#23`) —
needs clear UI labelling.

---

## 4. Feature 3 — Receive-with-discrepancy / short-delivery reports

An append-only discrepancy record against a delivered order, filed by one
party and approved/declined by the other, faithful to Itemcount's two-track
`discrepancyReport`.

### 4.1 Itemcount evidence

Itemcount's procurement domain (§3.4, OBS) carries a "two-track discrepancy
subsystem (filed vs received, with approve/decline/undo, photo + signature)"
on the PO aggregate, with per-user SoD grants for "create / approve / receive
/ file-discrepancy / approve-discrepancy" and a self-approve escape hatch.
§3.3 shows the group-PO lifecycle "place→(self)approve→receive→discrepancy→
resolve→complete→pay→close with append-only `PurchaseOrderHistory`", and §1
lists realtime events "discrepancy open/close/reopen". §5.2 names this
exactly as item **#2**: "Receive-with-discrepancy loop — today the order
jumps placed→confirmed→delivered; adding a lightweight discrepancy/received-
findings step (Itemcount's `discrepancyReport`) closes the biggest
correctness gap for real distribution." Itemcount stores the discrepancy as
its own record off the PO with append-only history — which is what lets county
honour immutability. The two tracks (filed vs received) map onto county's two
actors: reseller files what was received, wholesaler approves/declines (or
vice-versa).

### 4.2 county current state

county models a sell-side order only. The status machine is four states and
four status transitions in `OrderRules.java`; the spec's fifth transition,
delivered-unpaid to paid, is handled in `Orders.transition`, because `paid`
is a separate boolean set at or after deliver. `Orders.transition` drives
confirm/cancel/deliver/mark-paid;
`order_line` copies product_name/unit/quantity/price/total at placement and is
never updated. There is no discrepancy entity, table, service, DTO, or
endpoint; a short delivery has nowhere to be recorded. Migrations stop at
`V2__owner_identities.sql`.

### 4.3 Scope status — deferred by ADR 0012 (explicit)

ADR 0012, Decision, verbatim: "There is no edit endpoint. Short deliveries
and other differences between what was ordered and what was delivered are a
discrepancy report, deferred past v1." Its Consequences close: "Revisit
together with discrepancy reports, when a pilot customer's short deliveries
need recording." The functional spec mirrors this in §12 ("| Wholesaler edits
and short-delivery discrepancy reports | Cancel and reorder covers the demo
(ADR 0012). |"); its §2 names only the no-edit rule. This is the only one of
the five deferred by a named
ADR, and that ADR anticipates its return — the strongest build mandate here.

### 4.4 Proposed Flyway model

`V3__discrepancy.sql`.

```sql
CREATE TABLE discrepancy_report (
  id              BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id       BIGINT NOT NULL REFERENCES business(id),
  order_id        BIGINT NOT NULL,
  filed_by        VARCHAR(8) NOT NULL CHECK (filed_by IN ('owner','reseller')),
  filed_at        TIMESTAMPTZ NOT NULL,
  status          VARCHAR(8) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','approved','declined')),
  note            VARCHAR(500) NOT NULL DEFAULT '',
  resolution_note VARCHAR(500) NOT NULL DEFAULT '',
  resolved_by     VARCHAR(8) CHECK (resolved_by IN ('owner','reseller')),
  resolved_at     TIMESTAMPTZ,
  shortfall_total NUMERIC(12,2) NOT NULL,     -- captured sum of line shortfalls
  idempotency_key UUID NOT NULL,
  FOREIGN KEY (tenant_id, order_id) REFERENCES purchase_order(tenant_id, id),
  UNIQUE (tenant_id, order_id),               -- one report per order
  UNIQUE (tenant_id, idempotency_key),
  UNIQUE (tenant_id, id)
);
CREATE TABLE discrepancy_line (
  id                BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id         BIGINT NOT NULL,
  report_id         BIGINT NOT NULL,
  product_id        BIGINT NOT NULL,
  product_name      VARCHAR(80) NOT NULL,     -- copied at filing, like order_line
  unit              VARCHAR(4) NOT NULL,
  ordered_quantity  INTEGER NOT NULL,         -- copied from order_line.quantity
  received_quantity INTEGER NOT NULL CHECK (received_quantity BETWEEN 0 AND 9999),
  price             NUMERIC(12,2) NOT NULL,   -- copied from order_line.price
  shortfall_total   NUMERIC(12,2) NOT NULL,   -- (ordered-received)*price; negative = over-delivery
  FOREIGN KEY (tenant_id, report_id)  REFERENCES discrepancy_report(tenant_id, id),
  FOREIGN KEY (tenant_id, product_id) REFERENCES product(tenant_id, id),
  UNIQUE (report_id, product_id)              -- mirrors order_line UNIQUE(order_id,product_id)
);
CREATE INDEX discrepancy_open
  ON discrepancy_report (tenant_id, status, filed_at DESC);
```

`discrepancy_line` FKs to `product(tenant_id, id)` — **not** `order_line`,
which has no `(tenant_id, id)` unique key to target (its only unique is
`(order_id, product_id)`); identity-by-product is sufficient since an order
has at most one line per product. `ordered_quantity`/`price`/`name`/`unit`
are captured copies, so the record is immutable even if the product is later
edited/deactivated. New entities `DiscrepancyReport` + `DiscrepancyLine`
mirror `PurchaseOrder`/`OrderLine`.

### 4.5 REST endpoints

Reseller (`/order/{token}`): `POST .../orders/{id}/discrepancy` (file,
201 / 200 replay), `GET .../orders/{id}/discrepancy` (404 if none),
`POST .../orders/{id}/discrepancy/approve`,
`POST .../orders/{id}/discrepancy/decline` (resolve an owner-filed report).

Owner (`/`): `POST /orders/{id}/discrepancy` (file, 201 / 200 replay),
`GET /orders/{id}/discrepancy`, `POST /orders/{id}/discrepancy/approve`,
`POST /orders/{id}/discrepancy/decline` (resolve a reseller-filed report),
`GET /discrepancies?status=open` (dashboard list, polled every 30s per ADR
0004; no realtime).

### 4.6 Views / Inputs

New only. `Views.Discrepancy(long id, long orderId, long orderNumber, String
filedBy, String filedAt, String status[enum open|approved|declined], String
note, String resolutionNote, String resolvedBy?, String resolvedAt?,
BigDecimal shortfallTotal, List<Views.DiscrepancyLine> lines)`;
`Views.DiscrepancyLine(long productId, String productName, String unit, int
orderedQuantity, int receivedQuantity, BigDecimal price, BigDecimal
shortfallTotal)`; `Views.DiscrepancySummary(id, orderId, orderNumber,
resellerId, filedBy, status, shortfallTotal, filedAt)` for `GET
/discrepancies`. `Inputs.Discrepancy(@NotEmpty @Size(max=50) List<@NotNull
@Valid Inputs.DiscrepancyLine> lines, @Size(max=500) String note, @NotNull
UUID idempotencyKey)`; `Inputs.DiscrepancyLine(@NotNull @Positive Long
productId, @NotNull @Min(0) @Max(9999) Integer receivedQuantity)`;
`Inputs.Resolve(@Size(max=500) String note)`. Existing records unchanged
(no change to order views, `Place`, `Deliver`). Resource methods carry
`@APIResponse` 200 (replay) + 201 (created) with `Views.Discrepancy`,
matching `ResellerResource.place`.

### 4.7 Tenant scoping

tenant_id is never read from a request field. Owner endpoints resolve via
`subject()` → `Owners.tenant(subject).id`; reseller endpoints via
`Orders.reseller(token, lock)` → `reseller.tenantId` and `reseller.id`.
Report/line rows carry tenant_id and are always queried with `tenantId = ?`;
cross-tenant → 404. On the reseller side the report is additionally scoped to
`resellerId` through `purchase_order` (fetched with `PurchaseOrder.inTenant(
tenant, id, resellerId, lock)`), so one reseller cannot file against or read
another reseller's order. `filed_by`/`resolved_by` derive from which
authenticated path handled the call, never from the body.

### 4.8 Lock order and immutability (ADR 0012/0013)

Fits reseller → business → products → order without a new lock root. Products
are **not** locked: lines copy ordered_quantity/price from the immutable
`order_line` snapshot, so the live product price is irrelevant. File-by-
reseller takes `Reseller.byToken(token, PESSIMISTIC_WRITE)` then
`PurchaseOrder.inTenant(tenant, id, resellerId, PESSIMISTIC_WRITE)`. File-by-
owner and all approve/decline paths take a single `PurchaseOrder.inTenant(...,
PESSIMISTIC_WRITE)` (no business write-lock, nothing touches
`next_order_number`). The report/line rows are children read/written only
while the order's write lock is held, so the order lock is the serialization
point — the filing race is serialized by the order lock and backstopped by
`UNIQUE (tenant_id, order_id)` → 409 via the 23505 mapping. The idempotency
key is checked under the order lock. That differs from placement, which
checks it under the business lock: the order lock serialises only filings
against the same order, so a key reused across two orders is caught by the
unique constraint on the key and surfaces as the generic 409.

Fully respects ADR 0012: the discrepancy is an append-only sibling record,
never an edit of the order. The order's lines, note, delivery_date and total
are never touched; its status stays `delivered` and `paid` stays as set.
`discrepancy_line` captured values never change after filing; resolution only
sets status + resolved_by + resolved_at + resolution_note (analogous to how
an order changes only status + paid). A short delivery does **not** recompute
the order total or create any money adjustment — the `paid` boolean remains
the v1 money model; `shortfall_total` is a purely informational captured
figure (any credit/receivable is itself a Beyond-v1 item). This is the
precise revisit ADR 0012 anticipated, implemented as record-not-edit so the
cancel-and-reorder path and the immutable production run are untouched.

### 4.9 Tests first

- **Happy path:** deliver, reseller files one short line → 201; GET returns
  captured ordered/received/price/shortfall; order total/status/paid
  unchanged afterwards.
- **Idempotency:** same reseller re-POSTs the same key → 200 replay, no
  second row; different reseller reusing the key → 409.
- **Concurrency:** two simultaneous file calls on one order → exactly one
  succeeds (order lock), the other 409s from `UNIQUE (tenant_id, order_id)`.
- **Tenant isolation:** owner B cannot GET/approve/decline a report on
  business A's order → 404.
- **Reseller isolation:** reseller R2's token cannot file/view a report on
  R1's order → 404.
- **Status guard:** filing on a placed/confirmed/cancelled order → 409
  "Order is now <status>" (only delivered orders accept a discrepancy).
- **Actor rule:** the filer cannot resolve their own report; a reseller-filed
  report is resolved only by the owner and vice-versa; resolving an already-
  resolved report → 409.
- **Computation:** `shortfall_total = (ordered-received)*price` per line and
  summed; over-delivery (received>ordered) yields a negative shortfall and is
  accepted; received==ordered on every line is rejected 422 or accepted as
  zero-shortfall — pin the decision in the test.
- **Validation:** productId not on the order → 422; duplicate productId lines
  → 422; receivedQuantity outside 0..9999 → 422.
- **OpenAPI:** new discrepancy views + paths present with 200+201; drift
  check passes.

### 4.10 Effort and recommendation

Effort **M**. Build third. It is the highest-correctness increment (§5.2 #2)
and ADR 0012's own revisit trigger, low-risk because it reuses placement's
lock/idempotency/capture patterns and touches no existing contract. But it
adds no demo value until a customer must record shortfalls, so it should not
pre-empt price tiers. Build as record-not-edit exactly as specified; defer
any money/credit effect.

**Risks.** (1) Stakeholders may expect an approved short-delivery to reduce
what the reseller owes, but v1's money model is a single `paid` boolean and
ADR 0012 forbids changing the order total — `shortfall_total` is
informational; a real receivable/credit pulls in the deferred payments slice
and must be a separate record. (2) One-report-per-order diverges from
Itemcount's reopen/multiple-report model; if reopen is later needed it wants
a status history or per-order sequence rather than a unique constraint.
(3) Lines are matched by `product_id` (order_line has no `(tenant_id, id)`
unique key) — correct today, but a future multi-line-per-product change
breaks the assumption. (4) SoD: Itemcount separates file vs approve by grant;
county has one owner per business, so self-approval is prevented only by the
counterparty rule, acceptable until multi-user (also deferred). (5) No
realtime (ADR 0004): owners see new discrepancies only on the 30s poll.

---

## 5. Feature 4 — Minimum order quantity and order-create windows

Per-product MOQ + per-delivery-date MOQ tally, plus an order-window
start/end time and restricted days of week, translating Itemcount's
`moqTallyTotalOfSameDeliveryDate` / `createOrderStartTime`/`EndTime` /
`restrictedDayOfWeek`.

### 5.1 Itemcount evidence

Report §3.3, under *Rich workflows*: "MOQ/batch order windows
(`createOrderStartTime`/`EndTime`, `restrictedDayOfWeek`,
`moqTallyTotalOfSameDeliveryDate`)". This sits inside the
BusinessGroup→BusinessGroupMember wholesaler→reseller relationship that county
collapses into `Reseller.token` + `/order/{token}`. §5.2 item **#4** grounds
the translation: "Minimum order quantity / order windows — Itemcount's MOQ +
`createOrderStartTime`/`EndTime`/`restrictedDayOfWeek`; county already models
cutoff/delivery days, so this is an extension, not new infrastructure." (INF:
the bundle shows field names only; `moqTallyTotalOfSameDeliveryDate` implies
MOQ is satisfied against the cumulative total of a member's same-delivery-date
orders, not per single order.)

### 5.2 county current state

county already models order timing but has no MOQ and no create-side window.
`business.cutoff` (TIME) + `business.delivery_days` (INTEGER[], CHECK
cardinality>0; the ISO 1-7 range is enforced in `Inputs.Business`, not in
the database); `product.price` flat, no minimum; `order_line.quantity`
INTEGER CHECK BETWEEN 1 AND 9999; `purchase_order.delivery_date` DATE,
indexed by `order_delivery` (`reseller_recent` is on `placed_at`).
`DeliveryDates.compute(now,
zone, cutoff, days)` is a pure function picking the delivery date; `Orders`
uses `ZONE = Asia/Manila`. `OrderRules` governs only the status machine.
There is no create-time window, restricted-create-day, or per-product/per-date
minimum anywhere.

### 5.3 Scope status — MOQ deferred by functional spec; the window half is adjacent/new

functional-spec §2: "Not in v1, by decision: ... price tiers, minimum order
quantities ..."; §12: "| Price tiers, per-reseller price lists, minimum order
quantities | Every reseller pays the same in the reference customer. |". The
v1 design §*Out of scope* also lists "minimum order quantities". So MOQ is
explicitly deferred. The order-create **window** half (`createOrderStartTime`/
`EndTime`, `restrictedDayOfWeek`) is **not** named anywhere in the spec as
deferred — it is adjacent/new; county's existing `cutoff` + `delivery_days`
are the v1 analogue on the *delivery* side only (`restrictedDayOfWeek` is the
create-side inverse).

### 5.4 Proposed Flyway model

No new tables, no new FKs — additive columns plus one tenant-scoped index.
`V3__order_windows_moq.sql`:

```sql
ALTER TABLE business
  ADD COLUMN order_window_start TIME,            -- null = no create-time lower bound
  ADD COLUMN order_window_end   TIME,            -- null = no create-time upper bound
  ADD COLUMN restricted_days    INTEGER[] NOT NULL DEFAULT '{}',  -- ISO dow blocked for placing
  ADD COLUMN min_order_value    NUMERIC(12,2) NOT NULL DEFAULT 0; -- MOQ tally per reseller per delivery_date
ALTER TABLE business
  ADD CONSTRAINT business_order_window_order
    CHECK (order_window_start IS NULL OR order_window_end IS NULL
           OR order_window_start <= order_window_end),
  ADD CONSTRAINT business_restricted_days_range
    CHECK (restricted_days <@ ARRAY[1,2,3,4,5,6,7]
           AND cardinality(restricted_days) < 7),   -- can't lock out all 7 days
  ADD CONSTRAINT business_min_order_value_nonneg CHECK (min_order_value >= 0);

ALTER TABLE product
  ADD COLUMN min_order_quantity INTEGER NOT NULL DEFAULT 1
    CHECK (min_order_quantity BETWEEN 1 AND 9999);  -- mirrors order_line.quantity range

-- supports the per-reseller per-delivery-date MOQ tally read inside place()
CREATE INDEX reseller_delivery
  ON purchase_order (tenant_id, reseller_id, delivery_date, status);
```

Defaults make the migration a no-op for existing tenants (window null =
always open, restricted_days empty, min_order_value 0, min_order_quantity 1)
— expand-first per ADR 0009. `restricted_days` mirrors `delivery_days`'
INTEGER[] style but allows empty, and adds a database range check that
`delivery_days` does not have. New columns map onto the existing
`Business`/`Product` entities as `Integer[]`/`BigDecimal`/`LocalTime` fields.

### 5.5 REST endpoints

All additive; no new paths. Owner: `PUT /settings` and `POST /tenants` gain
`orderWindowStart`, `orderWindowEnd`, `restrictedDays`, `minOrderValue`;
`GET /settings` returns them, and so does every other response built from
`Views.Business`: `POST /tenants` and the `tenant` inside `GET /me`.
`POST`/`PUT /products` gain `minOrderQuantity` and `GET /products` returns
it. Reseller: `GET /order/{token}` —
`Views.Catalog` gains `orderingOpen` (bool), `opensAt`/`closesAt` copy,
`minOrderValue`, and per-product `minOrderQuantity` on `CatalogProduct` so the
form can disable submit and show "opens at 8:00 AM". `POST /order/{token}/
orders` — `Orders.place()` now enforces window + restricted-day + per-product
MOQ + MOQ-tally and returns 422 on violation (same `ResellerResource`
endpoint).

### 5.6 Views / Inputs

Additive only. `Inputs.Business` gains `@Schema(format="time", nullable=true)
String orderWindowStart`, `String orderWindowEnd`, `Set<@NotNull @Min(1)
@Max(7) Integer> restrictedDays` (not `@NotEmpty` — unlike
`deliveryDays`), `@DecimalMin("0") @Digits(integer=10, fraction=2) BigDecimal
minOrderValue`. `Inputs.Product` gains `@Min(1) @Max(9999) Integer
minOrderQuantity`. Omission must not clear stored values, for the reason
§2.5 gives for tiers: `Owners.apply`/`product` overwrite these bodies
wholesale, and a dashboard that predates the slice (or one still running
during the rollout) would otherwise reset the owner's restricted days,
minimum order value and every product minimum on each edit. So on
`PUT /settings` and `PUT /products/{id}` a null (absent) `restrictedDays`,
`minOrderValue` or `minOrderQuantity` leaves the stored value unchanged;
the defaults (empty set, 0, 1) apply only on `POST /tenants` and
`POST /products` and in the migration. An explicit empty set, `0` or `1`
clears the setting. **Open point:** `orderWindowStart`/`orderWindowEnd` are
nullable with null meaning "open all day" (§5.9), so null cannot also mean
"unchanged" for them, and a client that predates the slice would clear the
window on each settings edit. Whether the window moves to a dedicated
endpoint (as tier assignment did) or takes an explicit sentinel is the
owner's choice. `Views.Business` gains
`orderWindowStart`/`orderWindowEnd` (nullable time strings; `Times.clock`
is not null-safe today, so it gains a null check),
`List<Integer> restrictedDays`, `BigDecimal minOrderValue`. `Views.Product`
and `Views.CatalogProduct` gain `int minOrderQuantity`. `Views.Catalog` gains
`boolean orderingOpen` plus nullable `opensAt`/`closesAt` and
`minOrderValue`, so the server-rendered reseller form (ADR 0001, minimal
client JavaScript)
can disable Place order and show when it reopens. `Views.Order`,
`Views.OrderSummary`, `Views.RecentOrder`, `Views.ProductionRun` **unchanged**
— no per-order persistence of these gates, preserving ADR 0012.

### 5.7 Tenant scoping

All new config is written only by the owner through the subject-derived tenant
(`Owners.settings`/`create`/`product`); all enforcement reads happen inside
`Orders.place()` where the tenant comes from the reseller token. No window/MOQ
value is ever read from a request field — the reseller's `Inputs.Place`
(lines, note, idempotencyKey) carries none of these. The MOQ-tally query is
scoped `tenantId = reseller.tenantId AND resellerId = reseller.id`.

### 5.8 Lock order and immutability (ADR 0012/0013)

Fits reseller → business → products → order with **zero new lock targets.**
(1) The window / restricted-day check is a pure time-of-day test (new domain
helper `OrderWindow.isOpen(now, ZONE, start, end, restrictedDays)`, modelled
on `DeliveryDates`) — no lock. (2) Per-product MOQ is checked inside the
existing ascending-product loop where the product `PESSIMISTIC_READ` lock is
already held (next to the existing active-product 422). (3) The MOQ-tally is a
single aggregate read over `purchase_order` (`sum(total)` where
tenantId+resellerId+deliveryDate and status<>'cancelled') executed **after**
the business `PESSIMISTIC_WRITE` is taken — because that business-row lock
already serializes every placement in the business, two same-reseller same-
delivery-date orders cannot both read a stale tally, so the gate is race-free
with no extra lock. New index `reseller_delivery` backs the read. Keep all
three checks short and allocation-free (no network calls under lock).
**Critical ordering within `place()`:** the idempotency replay short-circuit
must stay **before** the new gates, so a retried submit during a now-closed
window still returns the original order (HTTP 200), never a fresh 422.

Clean under ADR 0012. All four checks are placement-time **gates** only;
nothing is written onto `purchase_order` or `order_line` for them.
Consequences to accept and document: (a) the MOQ-tally is evaluated once, at
placement, against the cumulative not-cancelled total for that
reseller+delivery_date including the new order; if a sibling order is later
cancelled and drops the running total below `min_order_value`, the surviving
order is **not** retroactively invalidated — immutability wins, the gate only
blocks new placement. (b) `min_order_quantity` constrains only the quantity at
placement; existing lines keep their quantity if the owner later raises the
minimum. (c) lowering/raising any limit affects new orders only. Add to
functional-spec §5.7:

| Change | Existing orders | New orders |
|---|---|---|
| Order window / MOQ / restricted days changed | unchanged | gated with new settings |

### 5.9 Tests first

- **`OrderWindow` domain unit tests** (like the delivery-date cases in
  `DomainRulesTest`): open/closed at start-1min, start, end, end+1min in
  Asia/Manila; restricted weekday
  blocks regardless of time; null start/end = open all day; validation
  rejects start>end and restricted_days covering all 7.
- **Per-product MOQ:** quantity < `product.min_order_quantity` → 422 naming
  the product and minimum; quantity == minimum succeeds; default 1 leaves
  existing behaviour unchanged.
- **MOQ tally:** first order below `min_order_value` → 422; an order bringing
  the reseller's same-delivery-date cumulative to ≥ min succeeds; a tiny top-
  up succeeds once cumulative ≥ min; cancelled siblings excluded.
- **Window race** (mirrors the deactivated-product-between-render-and-submit
  test): catalog rendered while open, window closes before submit →
  `place()` 422; `orderingOpen` flag reflects current state.
- **Idempotent replay bypasses the gates:** a repeated submit with the same
  key during a now-closed window still returns the original order as 200.
- **Tenant isolation:** business B's window/MOQ never affect business A's
  resellers; confirm no window/MOQ field is honoured from the reseller body.
- **Concurrency:** two same-reseller same-delivery-date placements racing the
  tally gate serialize under the business lock with no under-min slip-through;
  a `min_order_quantity` edit racing a placement.
- **PUT without the new fields:** `PUT /settings` and `PUT /products/{id}`
  bodies that omit `restrictedDays`, `minOrderValue` and `minOrderQuantity`
  keep the stored values; an explicit empty set, `0` or `1` clears them.
- **OpenAPI drift:** additive fields on `Views.Business`/`Product`/
  `CatalogProduct`/`Catalog` and `Inputs.Business`/`Product`; order views
  unchanged.

### 5.10 Effort and recommendation

Effort **M**. Build fourth, as a fast follow after discrepancy (or swap with
it — both are M and independent). MOQ/windows is §5.2 item 4 and the reference
customer charges every reseller the same, so there is no v1 urgency; but it is
a cheap, self-contained M that reuses `cutoff`/`delivery_days` infrastructure
with no new tables, endpoints, or locks.

**Risks.** (1) Tally semantics are INF from the bundle: confirm with the
owner that MOQ is cumulative-per-delivery-date-per-reseller (as designed) vs a
simple per-order minimum — it changes the `place()` check and the reseller
copy. (2) A closed window returns 422 with no way forward unless the catalog
exposes `opensAt` — ship the `Views.Catalog` window fields in the same slice.
(3) `restricted_days` is a footgun; the `cardinality<7` CHECK guards the hard
case but a near-total restriction wants a dashboard warning. (4) Pick 422 (as
designed, matching the deactivated-product rejection) over 409 and stay
consistent. (5) Keep every time-of-day comparison in ZONE, never system time.
(6) Must not re-run gates on idempotent replay (covered in tests).

---

## 6. Feature 5 — Online payment link / collectible ordering

A Xendit `createPaymentLink` equivalent, turning the single boolean `paid`
flag into a real collection workflow (payment-link issuance + a signature-
verified provider webhook), with the external Xendit HTTP client kept as a
clearly separated, stubbable boundary.

### 6.1 Itemcount evidence

Report §2 *Payments / money rails*: "Xendit is primary (invoice/payment
links, sub-accounts, payouts…); PayPal for subscriptions; GCash both as a POS
tender and a supplier payout account; Stripe named but 0 operations
observed." §3.4: a reseller order is a linked PurchaseOrder/SalesOrder pair
with "Public unauthenticated document routes" and "Payment via Xendit
`createPaymentLink`." §3.5: A/R `Invoice` with
`SetInvoiceStatusToPaid`/`SendInvoiceEmail`, Xendit payouts+subaccounts, and
OBS parallel status fields incl. `paymentStatus`/`customerCollectionStatus`
separate from order status. §5.2 item **#5** (verbatim): "Online payment link
— county has only a boolean `paid`; a Xendit `createPaymentLink` equivalent
turns it into a collectible-ordering tool." (INF: the collect-before-deliver
separation of paymentStatus from fulfilment status.)

### 6.2 county current state

No payment entity, no external integration. On `purchase_order`:
`paid BOOLEAN NOT NULL DEFAULT FALSE` and `paid_at TIMESTAMPTZ`. Collection is
two paths in `Orders.transition(...)`: `deliver` records `paid`/`paidAt` from
`Inputs.Deliver.paid`, and target `"paid"` (`POST /orders/{id}/mark-paid`)
guards `if (!current.equals("delivered") || order.paid) throw new Problem(409,
...)`. `paid` is deliberately out of the `OrderRules` state machine. The wire
exposes `paid` on both order views and `paidAt` on `Views.Order` only. A
reseller already sees `paid`/`paidAt` on their own order
(`GET /order/{token}/orders/{id}`); what is missing is any way to pay
online, for the reseller or anyone else.

### 6.3 Scope status — deferred past v1 by the functional spec

functional-spec §12, verbatim: "| Payments: GCash or Maya reference, partial
payments, receivables per reseller | The paid flag demos the idea. |".
Reinforced by §2: "Not in v1, by decision: ... payments beyond the flag ...".
The report §5.2 ranks it #5 of five, the lowest-leverage increment.

### 6.4 Proposed Flyway model

`V3__payments.sql`.

```sql
CREATE TABLE payment_link (
  id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id   BIGINT NOT NULL REFERENCES business(id),
  order_id    BIGINT NOT NULL,
  pay_token   VARCHAR(22) NOT NULL UNIQUE,            -- public credential, like reseller.token
  provider    VARCHAR(8) NOT NULL CHECK (provider IN ('xendit','manual')),
  amount      NUMERIC(12,2) NOT NULL CHECK (amount > 0),  -- captured = order.total; never recomputed
  status      VARCHAR(8) NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','paid','expired','failed','void')),
  external_ref VARCHAR(64),                            -- Xendit payment-link / invoice id
  external_url TEXT,                                   -- hosted checkout URL
  created_at  TIMESTAMPTZ NOT NULL,
  expires_at  TIMESTAMPTZ,
  paid_at     TIMESTAMPTZ,
  FOREIGN KEY (tenant_id, order_id) REFERENCES purchase_order(tenant_id, id),
  UNIQUE (tenant_id, id),
  UNIQUE (provider, external_ref)                     -- the webhook looks a link up without a tenant
);
-- at most one live link per order (retries allowed only after the prior leaves 'pending')
CREATE UNIQUE INDEX one_pending_link
  ON payment_link (tenant_id, order_id) WHERE status = 'pending';
CREATE INDEX payment_link_order ON payment_link (tenant_id, order_id);

CREATE TABLE payment_event (                           -- append-only webhook ledger
  id                BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tenant_id         BIGINT NOT NULL REFERENCES business(id),
  payment_link_id   BIGINT NOT NULL,
  order_id          BIGINT NOT NULL,
  provider          VARCHAR(8) NOT NULL,
  external_event_id VARCHAR(80),                       -- provider event id, for dedup; NULL on manual rows
  event_type        VARCHAR(24) NOT NULL,              -- 'paid','expired','failed'
  raw_status        VARCHAR(32) NOT NULL,              -- provider's status text; 'manual' on manual rows
  amount            NUMERIC(12,2) NOT NULL,
  received_at       TIMESTAMPTZ NOT NULL,
  FOREIGN KEY (tenant_id, payment_link_id) REFERENCES payment_link(tenant_id, id),
  FOREIGN KEY (tenant_id, order_id)        REFERENCES purchase_order(tenant_id, id),
  UNIQUE (tenant_id, provider, external_event_id)      -- webhook replay is a no-op
);
```

`purchase_order.paid` / `paid_at` stay the source of truth for "collected"; a
settled `payment_link` flips them (nothing else on the order changes). Both
existing manual paths, `mark-paid` and `deliver` with `paid=true`, are
re-expressed as a `provider='manual'` `payment_link` row with one
`payment_event`, so payments recorded after the release land in one ledger.
The migration backfills nothing, and the previous release keeps setting
`paid` without ledger rows during the rolling upgrade (ADR 0009 rule 2), so
`purchase_order.paid` stays authoritative for earlier and in-flight
payments.

A `payment_event` row exists only for an event that passed verification, so
the table carries no "verified" flag: an unverified callback writes nothing
(§6.9, risk 2). A manual row has no provider event, so its
`external_event_id` is NULL; PostgreSQL's unique constraint ignores NULLs,
and the order lock plus the `order.paid` guard keep manual rows single.
`external_ref` is unique per provider rather than per tenant, because the
webhook has to find the link before it knows the tenant (§6.7), as
`pay_token` and `reseller.token` already are.

### 6.5 REST endpoints

- `POST /orders/{id}/payment-link` — owner (JWT): issue (or return the
  existing pending) link for own order; 201 `Views.PaymentLink`.
- `GET /orders/{id}/payment-link` — owner: current link + status (404 if
  none).
- `POST /orders/{id}/mark-paid` and `POST /orders/{id}/deliver` with
  `paid=true` — owner: **keep**; each now also writes a `provider='manual'`
  `payment_link` + `payment_event` so offline collection joins the same
  ledger.
- `GET /order/{token}/orders/{id}/payment` — reseller (order token): sees
  their own order's link + status, scoped token→reseller→order.
- `GET /pay/{payToken}` — public (`pay_token` is the only credential, 404 for
  unknown/void, mirroring ADR 0003): minimal `Views.PaymentStatus` for the
  hosted-checkout return page.
- `POST /webhooks/xendit` — external machine callback (**not** subject/token
  scoped): verify Xendit `x-callback-token`/signature, resolve tenant from the
  stored `payment_link` via `external_ref`, dedup via `payment_event`, flip
  `payment_link.status` + `order.paid`.

### 6.6 Views / Inputs

New: `Views.PaymentLink(long id, long orderId, String provider, String
status, BigDecimal amount, String payUrl, @nullable String checkoutUrl,
String createdAt, @nullable String expiresAt, @nullable String paidAt)` with
`@Schema` enumerations for provider/status; `Views.PaymentStatus(long
orderNumber, String status, BigDecimal amount, @nullable String paidAt)` for
the public page (no reseller PII, no token echoed). `Inputs.PaymentLink(
@Pattern("xendit|manual") String provider, @Positive @Nullable Integer
expiresInHours)` — amount is derived from `order.total`, never accepted from
the client. Existing contracts stable: `Views.Order` keeps `paid`/`paidAt`
and `Views.OrderSummary` keeps `paid`, unchanged. The webhook body is provider-defined and is
**not** modelled as an `Inputs` record (it is parsed/verified, not bean-
validated as a trusted DTO). Errors via `Problem`: 404 unknown tokens, 409
already-paid/void, 422 validation.

### 6.7 Tenant scoping

tenant never comes from a request field. Owner endpoints derive tenant from
the Google subject; the reseller view from the order token. The two anonymous
surfaces resolve tenant from **server-stored rows**, not the request body:
`GET /pay/{payToken}` looks up `payment_link.pay_token` (carries tenant_id;
unknown → 404, probe-proof like reseller tokens), and `POST /webhooks/xendit`
first verifies the provider's signature/token, then finds the `payment_link`
by `external_ref` and reads tenant_id from it — the JSON body's own ids are
never trusted as tenant scope. `pay_token` is generated exactly like reseller
tokens (16 random bytes, `SecureRandom`, Base64url no padding = 22 chars;
the generator is the private `Owners.token()`, to be shared rather than
copied),
and like `reseller.token` is a bearer secret that must never be logged.

### 6.8 Lock order and immutability (ADR 0012/0013)

The canonical order is extended with `payment_link` as the **tail child after
order**: reseller → business → products → order → payment_link. Owner link
issuance locks the `purchase_order` row `PESSIMISTIC_WRITE` then inserts/locks
the `payment_link` — no reseller/business/product locks needed. The webhook
settlement path involves neither reseller nor products: it locks the
`purchase_order` row (`PESSIMISTIC_WRITE`) then the `payment_link` row, which
respects "order, then its children" and never acquires a child lock before its
parent. New locks: `payment_link` row only, always last. ADR 0013 allows
no network call while a lock is held, so issuance is two short transactions
around the provider call: the first locks the order and inserts the pending
link, the Xendit request runs with no transaction open, and the second
stores `external_ref` and `external_url` (or marks the link failed).
Concurrency
correctness also leans on the `one_pending_link` partial unique index and
`payment_event`'s `UNIQUE (tenant_id, provider, external_event_id)` so
concurrent deliveries settle exactly once.

Stays within ADR 0012: an order's lines, note, delivery_date and total never
change, and captured `order_line` prices are untouched — a settled
`payment_link` only flips the order's `paid`/`paid_at` (ADR 0012 explicitly
permits status + paid to change) plus writes its own payment rows, and
`amount` is captured at issuance from `order.total` and never recomputed.
**One deliberate rule relaxation to flag for owner sign-off:** today
`mark-paid` requires `status == 'delivered'`. Collectible ordering means
collecting before delivery, so the webhook settlement path should be allowed
to set `paid` when the order status is in {placed, confirmed, delivered} and
reject only `cancelled`. That is inside ADR 0012's decision (only `paid`
changes), but it amends accepted text elsewhere: functional-spec §5.6 allows
"delivered, unpaid → delivered, paid" and makes every other transition a
409, and ADR 0012's consequences count "four states and five transitions".
It should be a conscious decision recorded in the spec (§11), not silently
introduced.

The relaxation has a second half that is easy to miss. `deliver` assigns
`paid` and `paid_at` unconditionally from `Inputs.Deliver.paid`. Once an
order can be paid before delivery, `deliver` with `paid=false` would clear
a settled payment and `paid=true` would overwrite its `paid_at`. `deliver`
must leave an already-paid order's `paid`/`paid_at` untouched, with a test.

### 6.9 Tests first

- Owner issues a link for own order → 201 `Views.PaymentLink`, status
  pending, public `payUrl` present; issuing again returns the **same** pending
  link (`one_pending_link`), not a duplicate.
- **Isolation:** owner B cannot issue/read a link for owner A's order → 404;
  reseller token Y cannot read reseller X's order payment → 404;
  `GET /pay/{unknown}` and `/pay/{void}` → 404 (probe-proof, ADR 0003).
- **Webhook happy path:** valid signed Xendit 'paid' callback flips
  `payment_link.status='paid'` and `purchase_order.paid=true` + `paid_at`,
  writes one `payment_event`.
- **Webhook idempotency:** replaying the same `external_event_id` returns 200
  and makes no further state change.
- **Webhook auth:** bad/missing signature → 401 and zero state change: no
  `payment_event` row and no settlement.
- **Webhook amount mismatch** vs captured amount → rejected/flagged,
  `order.paid` stays false.
- **Concurrency:** two simultaneous webhook deliveries for one event settle
  exactly once (order row lock + event UNIQUE); webhook-vs-manual mark-paid
  race leaves `paid=true` with a single `paid_at`.
- **Domain rules:** settling a link whose order is 'cancelled' → 409;
  collecting on a 'placed'/'confirmed' order succeeds under the agreed
  collect-before-deliver relaxation; already-paid order → 409.
- **Manual mark-paid** and **deliver with `paid=true`** each record a
  `provider='manual'` link + `payment_event` without changing the existing
  200 contract.
- **Deliver after online payment:** delivering an order the webhook already
  settled keeps `paid=true` and the original `paid_at`, whatever
  `Inputs.Deliver.paid` says.
- **OpenAPI:** payment views appear, no entity leaks, `Views.Order` shape
  unchanged.

### 6.10 Effort and recommendation

Effort **M**. Build last, after the pricing/receivables groundwork, and gate
the live provider call on a real Xendit merchant account — but land the
county-side data model + manual-provider ledger + webhook shape early, because
they are self-contained and fully testable without any external calls. The
spec defers payments beyond the flag (§12) and the report ranks this #5, yet
the migration + webhook contract + the collect-before-deliver rule decision
are the parts that need design review, and folding today's `mark-paid` into a
proper `payment_link`/`payment_event` ledger is immediate, low-risk value that
the Xendit HTTP client later plugs into unchanged.

**Risks.** (1) The collect-before-deliver relaxation of the `mark-paid` guard
is a semantic change to an accepted rule — owner-approved, not slipped in.
(2) Webhook security is the whole ballgame: signature/callback-token
verification must happen before any DB write, and tenant must be resolved only
from the stored `payment_link` (never the callback body), or the endpoint
becomes a cross-tenant settlement oracle. (3) No realtime (ADR 0004): the
owner dashboard won't see 'paid' until its 30s poll, and the public `/pay`
return page may show 'pending' briefly — design it to read live status, not
assume settlement. (4) Partial payments / per-reseller receivables are **not**
in this slice; resist scope creep toward A/R aging. (5) External Xendit
integration needs secrets managed outside the chart (ADR 0011) and cannot be
live-tested here — keep the client behind an interface with a stub/manual
provider so CI stays Docker-free (ADR 0005) and makes no external calls. (6) `pay_token` is a
bearer secret with the same leak surface as `reseller.token` — never log it,
prefer short expiries.

---

## 7. Recommended sequencing

All five are migrations numbered `V3__*`, so whichever ships first takes `V3`
and the rest renumber (`V4`, `V5`, …) in build order; they do not conflict in
content, only in the Flyway version number. Build order:

1. **Price tiers (M).** §5.2 #1, highest value, slots onto catalog +
   placement with no new infrastructure. It also establishes the "price that
   isn't the flat `product.price`" primitive that quotes depend on. Adds no
   lock on the placement path: tier prices are guarded by the existing
   product lock; assignment adds a reseller → price_tier pair (§2.8).
2. **Quotes → order (L).** §5.2 #3. Build second so the quote line reuses the
   captured-price primitive from tiers rather than hard-coding a second
   pricing path. Reuses the token document + idempotent capture wholesale.
   Weakest scope mandate (spec omission) — confirm with the owner first.
3. **Receive-with-discrepancy (M).** §5.2 #2, highest correctness, and ADR
   0012's own revisit trigger. Independent of 1 and 2 — can move earlier if a
   pilot's short deliveries land before differentiated pricing is needed.
   Touches no existing contract; adds no new lock root.
4. **MOQ + order windows (M).** §5.2 #4. Cheap, self-contained, additive
   columns only, no new tables/endpoints/locks. A fast follow; independent of
   the others (swap freely with #3).
5. **Online payment link (M).** §5.2 #5, lowest leverage. Land the data model
   + manual ledger + webhook shape early (fully testable offline), gate the
   live Xendit call on a merchant account. Build after receivables thinking
   matures, since partial payments/receivables are a further slice.

**Dependencies.** Quotes → tiers (shared captured-price primitive) is the only
hard ordering dependency. Everything else is independent. Two features
extend the ADR 0013 lock chain: quotes (the quote in the order's slot) and
payments (`payment_link` as the tail child after order, §6.8). Price tiers
reuse the product lock on placement (assignment adds a reseller →
price_tier pair, §2.8), discrepancy hangs children off the order lock
without locking them, and MOQ/windows add no lock at all. Only one feature
(payments) needs anything outside the cluster (a Xendit account + secrets per ADR 0011); the rest are pure county.

---

## 8. Draft ADR

> Proposed for `adr/0014-begin-beyond-v1-build.md`. In the repo's ADR idiom.

# ADR 0014: Begin the Beyond-v1 build, pricing first

Date: 2026-11-23
Status: proposed

## Context

county v1 is a demo, not a pilot ([v1 design](../design/2026-10-16-v1-design.md)).
Its scope was drawn as the thinnest slice that runs the wholesaler→reseller
loop in front of a prospect. The
[functional spec](../design/2026-10-31-v1-functional-spec.md) §2 lists the v1
features as exhaustive and names, in §2 and §12 *Beyond v1*, the features
deferred by decision: price tiers and per-reseller price lists, minimum order
quantities, payments beyond the `paid` flag, and wholesaler edits / short-
delivery discrepancy reports.
[ADR 0012](0012-orders-are-immutable-after-placement.md) defers discrepancy
reports specifically ("Short deliveries ... are a discrepancy report,
deferred past v1") and asks to "revisit ... when a pilot customer's short
deliveries need recording."

A reverse-engineering study of the Itemcount ERP
(`itemcount-deep-rea.md` §5.2 "What county should build next") ranks five
post-v1 increments, all of which are faithful minimal analogues of Itemcount
capabilities and all of which fit county's existing conventions: REST not
GraphQL, Flyway-owned schema with composite `(tenant_id, id)` uniqueness and
tenant-scoped FKs, the ADR 0013 pessimistic lock order, typed view/input
DTOs, and immutability after placement. The design amendment *Beyond-v1 build
plan informed by Itemcount* works each one through to a migration, endpoints,
DTOs, lock-order impact and tests.

We need a decision to lift the v1 scope freeze for these five, in a stated
order, without reopening the decisions v1 made for good reasons.

## Decision

- Accept the five features in the amendment — per-reseller price tiers, quotes
  → order, receive-with-discrepancy, MOQ + order-create windows, and online
  payment link — as approved post-v1 work, to be built in that priority order.
  Price tiers go first; quotes build on the captured-price primitive tiers
  introduce.
- Each feature is implemented as its own `V3+__*.sql` migration under ADR 0009
  rules (forward-only, expand-first, no-op for existing tenants), with new
  root tables carrying `UNIQUE (tenant_id, id)` and every table carrying
  tenant-scoped composite FKs.
- The ADR 0013 lock order is extended, not broken: price tiers add no lock on the
  placement path, because a product's tier prices are guarded by that
  product's lock, and assignment adds a reseller → price_tier pair; quotes
  take the order's slot (reseller → business → quote → order); discrepancy
  hangs children off the order lock; MOQ/windows add no new lock. Payments
  add one lock target, the `payment_link` row, taken last after the order
  (reseller → business → products → order → payment_link). A follow-up
  amendment to ADR 0013 records the quote position, the reseller →
  price_tier pair and the `payment_link` tail before that code merges.
- ADR 0012 stands. No edit endpoint is added. Discrepancy reports are an
  append-only sibling record, not an order edit; a short delivery never
  recomputes an order total. Tier/quote/payment changes never mutate captured
  `order_line` prices; only `status` and `paid` ever change on a placed order.
- Two guard relaxations require explicit owner sign-off before their code
  merges, because each loosens an accepted rule: (a) the override `DELETE` in
  price tiers is the first delete in the codebase and is justified only as
  pricing config with no history; (b) collectible payments let the webhook set
  `paid` on a placed or confirmed order, loosening `mark-paid`'s current
  "delivered only" guard and requiring `deliver` to stop overwriting the
  `paid` flag of an order that is already paid.
- What deactivating a price tier does to resellers already assigned to it is
  decided by the owner before the tier code merges; the reference
  implementation leaves them on the tier's prices.
- Each feature ships tests first, covering the matrix ADR 0013 already
  mandates (concurrent same-key submissions, keys reused across resellers,
  rollback without a number gap, tenant isolation, edits racing a placement)
  plus its own immutability and resolution rules, and must pass the OpenAPI
  drift check.

## Consequences

- The v1 scope freeze is lifted for exactly these five, in this order. The
  functional spec §12 rows for price tiers, MOQ, payments and discrepancy
  reports move from *Beyond v1* to in-progress as each lands. The functional
  spec (§12, and §11 for accepted amendments) is updated at that point; this
  ADR stays as accepted, and a later ADR supersedes it if the decision
  changes.
- Quotes have the weakest mandate (the spec neither includes nor explicitly
  defers them; only the RE report places them post-v1), so quotes do not start
  until the owner confirms the increment. The other four rest on the spec's
  own deferral rows or, for discrepancy, on ADR 0012's revisit trigger.
- The money model stays the single `paid` boolean through all five.
  `shortfall_total` on a discrepancy and `amount` on a payment link are
  captured, informational figures; real receivables, partial payments and
  per-reseller A/R remain a separate, later slice (functional spec §12).
- The backend keeps one persistence style (Panache over Flyway, ADR 0013) and
  one public hostname with the API under `/api` (ADR 0010); the only new
  outside dependency is Xendit for the last feature, with secrets outside the
  chart (ADR 0011) and the client stubbed so CI stays Docker-free (ADR 0005)
  and makes no external calls.
- Revisit the order if a pilot's needs invert the ranking — a pilot recording
  short deliveries before it needs differentiated pricing would pull
  discrepancy ahead of tiers, which the dependency graph allows (only
  quotes→tiers is a hard dependency).
