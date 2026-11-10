# county v1: interface reference brief

Date: 2026-10-31
Status: Mobbin pass done on 2026-11-10, findings in section 7; Superdesign
pass pending. Feeds the
[interface design](https://github.com/jasoncalalang/county-architecture/blob/main/design/2026-10-31-v1-ui-design.md)

## 1. Why this exists

The interface design was drawn from the functional spec alone. Before the
screens are built, each one should be checked against shipped products
that solve the same problem, so that county borrows patterns resellers and
wholesalers already know. This brief names the comparables, the exact
searches to run on Mobbin, what to look for in each result, and the rule
for adopting or rejecting a pattern. Sections 2 to 6 were
written in a session where Mobbin was not reachable; section 7 holds what
Mobbin showed.

## 2. Comparables

Products that put a supplier's price list in a buyer's hand, or an order
queue in an operator's hand. Found by web search; the notes are what the
sources say, not what the screens look like.

| Product | Why it is comparable | What the sources say |
|---|---|---|
| Rekki (UK, restaurant to supplier) | A buyer orders from a supplier's list on a phone with no account set-up; ordering and chat are one thread | "Order and chat with all your suppliers in one app"; "start ordering with a new supplier instantly, no account forms"; built for "the heat of the kitchen" |
| Choco (Berlin, restaurant to supplier) | Same shape; its "order guide" is a per-buyer price list | Order guides "customised by category"; "view, edit and send orders"; reviewers complain of the same order arriving by email, text and app |
| Growsari (Philippines, sari-sari store supply) | The reference buyer: a Philippine neighbourhood store ordering stock on a phone | 300 to 1,000 distributor-priced SKUs; no minimum order; own delivery network; wallet and payments in the same app |
| GrabMerchant (Southeast Asia, merchant order queue) | The operator side: a queue of incoming orders with accept, ready and cancel, used on a phone at the counter | Orders tab with a store status bar; short order numbers suffixed by order type; a dedicated device that rings and auto-accepts; a third-party POS shows "Extend", "Cancel", "Ready" |
| B2B Wave (wholesale ordering) | Fast quantity entry on a long list | "Fast Quantity Entry" with a list view and plus and minus buttons |
| Shopify POS and admin (order management) | The vocabulary of order status badges | Order detail is "where merchants handle customer service, refunds or fulfilment"; Polaris Badge carries Fulfilled, Unfulfilled, Paid, Unpaid |

## 3. Searches to run on Mobbin

One row per county screen. Run the app searches first, then the pattern
searches, and save the screens that fit into one Mobbin collection named
"county v1".

| county screen | Mobbin app searches | Mobbin pattern and flow searches | Look for |
|---|---|---|---|
| Order page (reseller) | Rekki, Choco, Growsari, Pepper, Cut+Dry, Ula, Zepto | "quantity stepper", "order form", "place order", "reorder", "cart" on mobile web | How quantity is entered on a long list: stepper, bare input, or tap-to-add. Whether the total is sticky. How the delivery date is shown before ordering. |
| Confirmation (reseller) | the same apps | "order confirmation", "order placed", "order summary" | What the buyer sees after placing: number, date, lines, what to do next. Whether cancel is offered there. |
| Inactive link | | "link expired", "invite expired", "access revoked", "empty state" | Tone of the message; whether a contact action is offered. |
| Orders (dashboard) | GrabMerchant, foodpanda partner, Deliveroo Hub, Shopify, Square | "order management", "incoming orders", "accept order", "order queue", "status filter", "tabs" | Row or card layout for an order; where confirm and cancel sit; how new orders are made loud; how a list is filtered by status and date. |
| Order detail | the same | "order details", "activity log", "timeline" | Line table shape, status history, actions placement. |
| Production run | Square for Restaurants, Toast, Lightspeed, Shopify (picking) | "pick list", "prep list", "batch summary", "print" | A per-product totals table and a print affordance. |
| Resellers and link reveal | WhatsApp Business, Telegram, Notion, Figma, Slack | "invite link", "share link", "copy link", "contact list", "block user" | The reveal after creating a link: copy buttons, message template, warning copy. |
| Products | Shopify, Square, Loyverse | "product list", "inline edit", "toggle", "price list" | Inline price editing and an active toggle without a form page. |
| Settings and set-up | Square, Shopify, Deliveroo Hub | "business hours", "cut-off", "onboarding", "store setup" | How a cut-off time and delivery days are entered; the set-up form after sign-in. |
| Status badges | | "badge", "status", "chips" | Colour plus word pairs for placed, confirmed, delivered, cancelled, paid, unpaid. |

## 4. How to judge a pattern

Adopt a pattern from a reference only when all four hold:

1. It survives the order page's constraints: works with JavaScript off,
   no icon font, under 50 KB, inside Viber's browser.
2. It is already familiar to the reseller or wholesaler from an app they
   use daily (GrabMerchant, Shopee, GCash, Viber), which is the test for
   "they already know this".
3. It does not add a rule to the server. Delivery date, totals and
   transitions stay as the v1 design computes them.
4. It changes something the current design does, not something it already
   does the same way.

Write each adopted pattern into the interface design as a one-line
"Reference:" note under the screen it changes, with the app and screen
name. A rejected pattern gets no note.

## 5. Superdesign pass

Build on Superdesign only after the Mobbin pass, in this order, using the
county tokens in section 3 of the interface design and the demo data in
the functional spec:

1. Order page and confirmation, 390 px, because the reseller side has the
   most to learn from references and the tightest constraints.
2. Orders list and order detail, 1280 px and 390 px.
3. Production run, resellers with link reveal, products, settings.

Each Superdesign draft that is kept replaces the matching artboard on the
existing canvas, so there is one set of screens, not two.

## 6. Open questions for the research to settle

- Quantity entry: a bare numeric input (current design) or a stepper.
  Steppers cost taps on long lists; inputs cost accuracy on small phones.
- A sticky total on the order page: worth the optional script, or noise.
- New-order loudness on the dashboard: a badge count on the Orders tab, a
  sound, or nothing beyond the 30-second poll.
- Whether the confirmation page should offer cancel directly.

## 7. Findings from the Mobbin pass

Run on 2026-11-10: 62 searches across the ten rows of section 3. The 67
screens worth keeping (28 iOS, 39 web) are in the private Mobbin collection
[county v1](https://mobbin.com/collections/8ceab915-edda-487d-bed5-92c23fd4ca3f/mobile/screens).

### What Mobbin could and could not show

- Mobbin has iOS and web only. Its web captures are desktop width, so
  nothing shows an HTML form inside Viber's or Messenger's browser, and
  nothing shows Android.
- Not on Mobbin: Rekki, Choco, Growsari, Pepper, Cut+Dry, Ula, Zepto,
  GrabMerchant, foodpanda partner, Deliveroo Hub, Toast, Lightspeed,
  Loyverse, Viber and GCash. There is no merchant-side food order queue and
  no Philippine B2B ordering app in the evidence. B2B Wave, Grab and Lazada
  were not searched.
- On Mobbin and used: Shopee (the buyer app and its My Shop seller home),
  foodpanda and Deliveroo (buyer apps), Faire Wholesale, Shopify (web and
  iOS), Square, Messenger, WhatsApp, Telegram, Slack, Notion and Figma.
- Criterion 2 of section 4 was therefore tested against Shopee, foodpanda
  and Messenger. The Shopee screens are the Indonesian build.

### Adopted

Each is written into the interface design with its "Reference:" note.

| Screen | Pattern | Reference | Design section |
|---|---|---|---|
| Order page | The total line and the Place order button stick to the bottom as one block | Shopee, Confirm Order; Woolworths, Cart | 5, 7.1 |
| Confirmation | While the order is placed: one line on what happens next, and a Cancel order button | Shopee, Order Details; foodpanda, Can I cancel my order? | 5, 7.3 |
| Orders | A count of placed orders beside Orders in the navigation, on every page | Shopee, My Shop; Shopify admin, Orders | 4, 6.3 |
| Orders | The main row action at the right edge, Cancel before it | Shopee, My Purchases; Instacart, Order approvals | 4, 6.3 |
| Resellers | A line in the link reveal saying who can use the link | WhatsApp, New call link; Splitwise, Invite link | 4, 6.6 |

Two of these reach past the interface design. The navigation count needs
`GET /orders` to accept `status` without `date`, added to section 11 of
the functional spec. Cancel on the confirmation page is added to section
4.3 of the functional spec; the rule and the route already existed.

### The open questions of section 6

- **Quantity entry: keep the bare numeric input.** A stepper appeared in 14
  apps, Shopee among them, but every one held 1 to 3 units, and it needs
  script or a page load per tap. county orders are 10 or 20 packs, where
  typing is faster. The one wholesale app, Faire, keeps a number box
  between its minus and plus. Evidence: strong on steppers in consumer
  apps, thin on wholesale.
- **Sticky total: keep it, pinned with the button.** 13 ordering apps keep
  a total fixed beside the main button. All are native apps, so a fixed
  footer inside an in-app browser with the keyboard open still needs a
  device test.
- **New-order loudness: a count beside Orders, no sound.** Every seller
  reference that signals waiting work uses a number beside a label. The
  loud patterns seen (a modal, a countdown) all sat beside an acceptance
  deadline, which county does not have. Sound cannot be seen in a
  screenshot and the apps that ring are not on Mobbin.
- **Cancel on the confirmation page: yes, while placed.** No thank-you
  screen offered cancel (0 of 7), but county's confirmation URL is also the
  order's permanent page, and that is where Shopee and Faire put it.

### Confirmed as designed

The references match what the design already does, so nothing changed and
no note was added: status chips with counts including zeros; date arrows;
the freshness line; a status badge plus a paid badge; amber for waiting,
green for done, grey for cancelled (cancelled was red in no app); the
two-column order detail with a dated history list; the bottom toast; the
delivery banner with a cut-off sentence (HelloFresh); the receipt-shaped
confirmation; the bare inactive-link message (Whereby); Block asking and
Unblock not asking (Messenger); a single set-up form; a labelled Print
button.

### Not adopted, but worth a decision

These failed criterion 2 on the evidence, or are gaps the references
exposed rather than patterns to borrow. None is in the interface design.

- Nothing says what happens after Copy link or Copy message. Figma and
  Wispr Flow confirm with words that name what was copied.
- The production run carries no time. Two printouts for the same date
  cannot be told apart once another order is confirmed. Fresha and Semrush
  stamp their reports.
- The quantity field starts at 0. A digit typed before that 0 turns 1 into
  10. An empty field with a 0 placeholder avoids it. This is a reading of
  the design, not a Mobbin finding.
- On a phone product card, tapping bare text is the only way to edit. No
  phone reference showed a price typed inside a list row; Turo and alias
  put the word Edit beside the price.
- Below 900 px the design does not say how the order detail lines, the
  production run rows or the seven day checkboxes are laid out, and the
  phone order card has no placed time.
- Unpaid uses the cancelled grey. No app showed a rendered Unpaid badge;
  Shopify shows money still owed in orange.
- Reorder appeared once, on Faire's order card. It would suit daily repeat
  orders and needs server work, so it is a later slice.

### Canvas and record

The design canvas already shows two things this pass did not adopt and the
interface design does not record: a preview of the message under the link
box, and a hint under Cut-off time on set-up. Its confirmation artboard
still says cancelling is done from the price list. The Superdesign pass
should bring the canvas and the record back in line.
