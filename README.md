# Affirm Integration Demo — Direct API vs Virtual Card (VCN)

An interactive, self-contained mockup that walks through Affirm's two checkout
integrations side by side, with a toggle at the top to switch between them:

- **Direct API** — `affirm.checkout(obj)` → `affirm.checkout.open()` → Affirm
  returns a `checkout_token` to your `user_confirmation_url` → server-side
  **Authorize / Capture / Void / Refund** via the Transaction API.
- **Virtual Card (VCN)** — `affirm.checkout.open_vcn({ success, error, checkout_data })`
  → `success(card_details)` hands you a one-time virtual card you run through
  your existing payment processor.

Three columns: a storefront (shopper view + back-office order management),
the sequence of operations, and a live backend/API activity log with
illustrative request/response bodies.

## Simulation build

This is a **simulation** — there are **no live API calls and no credentials**.
Every response is canned, illustrative test data, so the full flow can be
demonstrated anywhere as a static page. Sandbox verification pin: `123456`.

## Run it

Just open `index.html` in a browser, or visit the hosted page.

## Reference

- [Affirm checkout overview](https://docs.affirm.com/developers/docs/affirm-checkout-overview)
- [Managing transactions](https://docs.affirm.com/developers/docs/managing-transactions)

---

_Built with Cursor (Claude)._
