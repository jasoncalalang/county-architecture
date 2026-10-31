# county v1: interface reference brief

Date: 2026-10-31
Status: working document for the Mobbin research and Superdesign pass;
feeds the
[interface design](https://github.com/jasoncalalang/county-architecture/blob/main/design/2026-10-31-v1-ui-design.md)

## 1. Why this exists

The interface design was drawn from the functional spec alone. Before the
screens are built, each one should be checked against shipped products
that solve the same problem, so that county borrows patterns resellers and
wholesalers already know. This brief names the comparables, the exact
searches to run on Mobbin, what to look for in each result, and the rule
for adopting or rejecting a pattern. It was written in a session where
Mobbin was not reachable; nothing below is a Mobbin finding yet.

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
