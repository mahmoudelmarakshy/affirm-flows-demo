---
name: affirm-integration-demo
description: >-
  Create, rebrand, preview, and deploy the Affirm Direct API + Virtual Card
  (VCN) integration demo (a single-file HTML mockup). Use when asked to build or
  update an Affirm checkout demo for a merchant, rebrand the affirm-flows-demo,
  edit its DEMO_CONFIG, change the cart / theme / IDs / sample card, add a step
  or flow, or deploy the Affirm flows demo to GitHub Pages.
---

# Affirm Integration Demo

A self-contained, single-file HTML demo of Affirm's two checkout integrations —
**Direct API** and **Virtual Card (VCN)** — with a storefront, a sequence
diagram, and a live API activity log. It is a **simulation**: no live API calls,
no credentials, canned responses. Use it to walk a merchant through an
integration.

## The template

The demo is one file: [`template/index.html`](template/index.html), bundled with
this skill. Everything a merchant would change lives in a single **`DEMO_CONFIG`**
object near the top of its `<script>`. **Prices are in dollars**; the engine
computes cents and keeps the storefront, the Affirm checkout object, and the API
logs in sync — `subtotal → discounts → tax → total` and the payment-plan amounts
are all derived. **Never hardcode a total.**

Canonical source / updates: https://github.com/mahmoudelmarakshy/affirm-flows-demo

## Workflows

### A. Rebrand an existing demo
1. Find the demo file (`index.html`; ask the user if ambiguous).
2. Edit **only** the `DEMO_CONFIG` block — map the user's merchant details onto
   the fields (see reference below). Leave the engine below it untouched.
3. **Preview** (see below) and confirm totals look right.

### B. New merchant demo from the template
1. Copy `template/index.html` to the target location (new folder or repo).
   Default to a new folder named after the merchant, e.g. `casper-demo/index.html`.
2. Edit `DEMO_CONFIG` for the merchant.
3. Preview, then deploy if asked.

### C. Add a step or a flow
The engine is a small state machine:
- `FLOWS.direct` / `FLOWS.vcn` — ordered step arrays (`actors`, `kind`, `label`, `desc`).
- `runStep(step)` — a `switch` on `step.kind` driving the storefront, modal, and
  log (`logApi`, `logEvent`, `logDivider`).
- `DOCS` — Affirm doc links; pass `doc: DOCS.x` to `logApi` / `logEvent` for any
  call that references Affirm docs.

To add a step: add a flow entry **and** a matching `runStep` `case`. To add a
flow: add a key to `FLOWS` plus a toggle button.

## `DEMO_CONFIG` reference

| Group | Fields | Controls |
|-------|--------|----------|
| `brand` | `shortName`, `storeName`, `accent`, `accentDark`, `affirmAccent`, `searchPlaceholder`, `subnav[]` | Header chip, store name, theme colors, nav |
| `cart.items[]` | `name`, `sku`, `qty`, `price`, `emoji` | Line items + checkout object `items` |
| `cart.adjustments[]` | `label`, `amount` (neg = discount), `code`, `checkoutName` | Summary rows + checkout object `discounts` |
| `cart` | `shipping` (0 = FREE), `tax`, `currency` | Shipping/tax rows + totals math |
| `ids` | `publicApiKey`, `orderId`, `transactionId`, `checkoutToken`, `checkoutId` | Identifiers in the logs (merchant `order_id` vs Affirm `id`/loan id) |
| `customer` | `name`, `address`, `phone_number`, `email` | `shipping`/`billing` in the checkout object |
| `card` | `number`, `cvv`, `expiration`, `cardholder_name`, `charge_ari`, `billing_address` | Sample VCN from the `success` callback |
| `sandboxPin`, `metadata`, `urls` | — | Verification pin, checkout `metadata`, Direct API confirm/cancel URLs |

## Preview

It is one static file — open it in a browser, or capture a headless screenshot to
verify your edits (macOS):

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless --disable-gpu --hide-scrollbars --window-size=1500,950 \
  --screenshot=/tmp/affirm-demo.png "file:///ABS/PATH/TO/index.html"
```

Then read `/tmp/affirm-demo.png`. Confirm the storefront total, the checkout
object `total`, and the "As low as" figure all agree (they should, since they're
computed). To screenshot a completed flow, inject a driver before `</script>`:
`window.addEventListener('load', () => setTimeout(() => goTo(STEPS.length - 1), 350));`

## Deploy (GitHub Pages)

1. Put the demo in a repo and push.
2. **Settings → Pages → Source: Deploy from a branch**, `main` / `/ (root)`.
3. Live at `https://<org-or-user>.github.io/<repo>/` in ~1 minute. Include an
   empty `.nojekyll` file so Pages serves it as-is.

Any static host works — it's one file.

## Publish internally (QuickHost, optional)

For sharing with the **Affirm team** (not external merchants), publish to QuickHost,
Affirm's internal SPCS static hosting behind Okta SSO. This is **optional** and is
**not required** to use this skill.

Don't reimplement the upload here — defer to the separate **`quickhost`** skill
(plugin `quickhost@affirm-builders`). Once it's installed, just say
"upload this to quickhost" / "update the quickhost demo". The demo is
self-contained, so no CDN inlining is needed.

One-time setup: `claude plugin install quickhost@affirm-builders` and
`brew install snowflake-cli`. (For external/merchant sharing, use GitHub Pages
above — QuickHost links only open for authenticated Affirm users.)

## Rules

- Keep it **single-file**. Don't split into separate JS/CSS unless asked.
- Edit `DEMO_CONFIG` for merchant changes; don't hand-edit totals or duplicate
  values into the HTML.
- It's a **simulation** — never add real API keys, tokens, or PII.
- When citing an Affirm call in the log, attach the matching `DOCS` link.

## Sample prompts

See [examples.md](examples.md) for a full set. A few:

- "Rebrand the demo for Casper — navy theme, Original Mattress (Queen) $1,095 and 2× Foam Pillow $65, 10% launch discount, tax $92.40."
- "Spin up a new Affirm demo for H&R Block in a `hrblock-demo/` folder and deploy it to GitHub Pages."
- "Change the sample VCN card and the merchant order_id in the demo."
- "Switch the default flow to VCN and make the cart a $2,400 electronics order."
- "Add a step to the Direct API flow showing an order webhook after capture."
