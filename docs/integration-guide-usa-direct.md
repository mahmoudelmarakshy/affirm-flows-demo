<!--
Affirm USA — Direct Checkout (Transactions API) integration reference
Source: "Affirm Integration Guide Generator" (QuickHost presentation 7a8aab1c),
config: Region USA + Direct Checkout. Captured 2026-06-16 for the affirm-flows-demo accuracy pass.
This is generated reference content (authoritative endpoints/payloads), not hand-authored.
-->

Your Affirm Integration Guide
USA
Direct Checkout
Direct API

Your backend exchanges a checkout token for a persistent loan ID and manages the full transaction lifecycle — authorize, capture, void, and refund — via the Transactions API.

Step 1 — Include Affirm.js

Add this snippet to the <head> of your global page template for USA. Use the sandbox script URL for testing; swap to production before going live.

HTML — Affirm.js loader
Copy
<!-- Affirm SDK -->
<script>
var _affirm_config = {
  public_api_key: "YOUR_PUBLIC_API_KEY",
  script: "https://cdn1-sandbox.affirm.com/js/v2/affirm.js",       // Sandbox (testing)
  // script: "https://cdn1.affirm.com/js/v2/affirm.js", // Production (swap before going live)
  locale: "en_US",
  country_code: "USA"
};
(function(m,g,n,d,a,e,h,c){
  var b=m[n]||{},k=document.createElement(e),
  p=document.getElementsByTagName(e)[0],
  l=function(a,b,c){return function(){a[b]._.push([c,arguments])}};
  b[d]=l(b,d,"set");
  var f=b[d];b[a]={};b[a]._=[];f._=[];
  b._=[];b[a][h]=l(b,a,h);
  b[c]=function(){b._.push([h,arguments])};
  a=0;
  for(c="set add save post open empty reset on off trigger ready setProduct".split(" ");
      a<c.length;a++) f[c[a]]=l(b,d,c[a]);
  a=0;
  for(c=["get","token","url","items"];a<c.length;a++)
    f[c[a]]=function(){};
  k.async=!0;k.src=g[e];
  p.parentNode.insertBefore(k,p);
  delete g[e];f(g);m[n]=b
})(window,_affirm_config,"affirm","checkout","ui","script","ready","jsReady");
</script>
<!-- End Affirm SDK -->
Step 2 — Direct Checkout: Open Affirm Modal

Build the checkout object from your cart data and open the Affirm modal. After approval, Affirm POSTs a checkout_token to your user_confirmation_url.

JS — Checkout object + open modal
Copy
// Build from your cart/order data — do not hardcode
affirm.checkout({
  merchant: {
    user_confirmation_url: "https://yoursite.com/affirm/confirm",
    user_cancel_url: "https://yoursite.com/affirm/cancel",
    public_api_key: "YOUR_PUBLIC_API_KEY",
    user_confirmation_url_action: "POST",
    name: "Your Store Name"
  },
  currency: "USD",
  shipping: {
    name: { first: "Jane", last: "Smith" },
    address: {
      line1: "100 Main St",
      line2: "",
      city: "Chicago",
      state: "IL",
      zipcode: "60601",
      country: "USA"
    },
    phone_number: "3125550100",
    email: "jane@example.com"
  },
  billing: {
    name: { first: "Jane", last: "Smith" },
    address: {
      line1: "100 Main St",
      line2: "",
      city: "Chicago",
      state: "IL",
      zipcode: "60601",
      country: "USA"
    },
    phone_number: "3125550100",
    email: "jane@example.com"
  },
  items: [
    {
      display_name: "Product Name",
      sku: "SKU-001",
      unit_price: 49900,  // in cents
      qty: 1
    }
  ],
  order: {
    merchant_order_id: "YOUR_INTERNAL_ORDER_ID",
    total: 49900,         // in cents
    tax_amount: 0,
    shipping_amount: 0
  }
});

// Open the Affirm checkout modal
affirm.checkout.open();
Token rules
Copy
checkout_token lifecycle rules:
- Short-lived: use it immediately upon receipt at user_confirmation_url
- Do NOT store it long-term
- Authorize with it once, then discard
- Store the returned loan_id / charge_id (format: XXXX-XXXX) as your permanent identifier
- All future API calls use loan_id / charge_id, NOT checkout_token
Step 3 — Authentication

All Transactions API calls use HTTP Basic Auth. The affirm_request helper below is reused for every lifecycle call. Never expose your private key in client-side code.

Python — Authentication (base request setup)
Copy
import requests
from requests.auth import HTTPBasicAuth

# Select your environment — swap to production before going live
API_BASE = "https://api.global.sandbox.affirm.com/api/v1"         # Sandbox (testing)
# API_BASE = "https://api.global.affirm.com/api/v1"    # Production

PUBLIC_KEY  = "YOUR_PUBLIC_KEY"
PRIVATE_KEY = "YOUR_PRIVATE_KEY"  # Server-side only — never in client code

def affirm_request(method, path, **kwargs):
    url = f"{API_BASE}{path}"
    headers = {
        "Content-Type": "application/json",
    }
    return requests.request(
        method,
        url,
        auth=HTTPBasicAuth(PUBLIC_KEY, PRIVATE_KEY),
        headers=headers,
        **kwargs
    )
Step 4 — Authorize

Exchange the short-lived checkout_token for a persistent loan_id / charge_id. Store this identifier — it is used for all future lifecycle calls. API reference →

Python — Authorize
Copy
def authorize_transaction(checkout_token, order_id):
    """
    Called at your user_confirmation_url when Affirm POSTs the checkout_token.
    Returns a persistent loan_id / charge_id (format: XXXX-XXXX).
    """
    response = affirm_request(
        "POST",
        "/transactions",
        json={
            "transaction_id": checkout_token,  # The short-lived token
            "order_id": order_id               # Your internal order ID
        }
    )
    response.raise_for_status()
    data = response.json()

    # Persist this identifier — use it for capture, void, and refund
    loan_id = data.get("id")  # Format: XXXX-XXXX
    return loan_id
Step 5 — Capture

Capture should align with OMS fulfillment events — not immediately after authorization. Capturing finalizes the loan and is only valid before a void. API reference →

Python — Capture
Copy
def capture_transaction(loan_id):
    """
    Triggered by your OMS fulfillment event.
    loan_id is the XXXX-XXXX identifier from authorization.
    Only valid before a void. Finalizes the loan.
    """
    response = affirm_request(
        "POST",
        f"/transactions/{loan_id}/capture"
    )
    response.raise_for_status()
    return response.json()
Step 6 — Void

Void cancels an authorized transaction before capture. Only valid PRE-CAPTURE — cannot void a captured transaction. Triggered by OMS cancellation events. API reference →

Python — Void
Copy
def void_transaction(loan_id):
    """
    Triggered by OMS cancellation before fulfillment.
    Only valid PRE-CAPTURE. Cannot void a captured transaction.
    """
    response = affirm_request(
        "POST",
        f"/transactions/{loan_id}/void"
    )
    response.raise_for_status()
    return response.json()
Step 7 — Refund (full and partial)

Refunds are used after capture. Both full and partial refunds are supported. Triggered by OMS return or adjustment events. API reference →

Python — Refund (full and partial)
Copy
def refund_transaction(loan_id, amount_cents=None):
    """
    Triggered by OMS return/adjustment event POST-CAPTURE.
    amount_cents=None performs a full refund.
    amount_cents=<value> performs a partial refund.
    """
    body = {}
    if amount_cents is not None:
        body["amount"] = amount_cents  # Partial refund in cents

    response = affirm_request(
        "POST",
        f"/transactions/{loan_id}/refund",
        json=body
    )
    response.raise_for_status()
    return response.json()
OMS Integration Guidance

Affirm payment lifecycle must be driven by your Order Management System (OMS), not front-end events.

OMS state machine
Copy
Order State          → Affirm Action
─────────────────────────────────────────────────────
Checkout confirmed   → POST /transactions (authorize)
OMS: Fulfilled       → POST /transactions/{id}/capture
OMS: Cancelled       → POST /transactions/{id}/void
                        (only if not yet captured)
OMS: Returned        → POST /transactions/{id}/refund
                        (only after capture)

Rules:
- OMS is the system of record for order status
- Never infer payment state from front-end events
- Route lifecycle calls based on stored payment_provider field
- Do not call Affirm APIs for non-Affirm orders
- Store payment_provider and provider_reference_id at authorization
Environment Reference

Quick reference for all URLs. Use sandbox for testing and development; production for live merchant traffic.

Sandbox & Production URLs
Copy
Region: USA

SANDBOX (testing):
  Affirm.js script:      https://cdn1-sandbox.affirm.com/js/v2/affirm.js
  Transactions API base: https://api.global.sandbox.affirm.com/api/v1

PRODUCTION (go-live):
  Affirm.js script:      https://cdn1.affirm.com/js/v2/affirm.js
  Transactions API base: https://api.global.affirm.com/api/v1

Key rules:
- Always match sandbox keys with sandbox endpoints
- Always match production keys with production endpoints
- Never mix environments
Going Live Checklist

Complete all items before switching to production.

Checklist
Copy
[ ] Swap Affirm.js script to production URL (see Step 1 code — uncomment production line)
[ ] Update to live public API key
[ ] Set API_BASE to production URL in server-side code (see Step 3)
[ ] Update to live private API key (server-side only)
[ ] End-to-end smoke test with a real transaction in production
[ ] Error tracking configured for Affirm interactions
[ ] Alerts configured for failure thresholds

Need a different configuration?

← Generate a new guide
