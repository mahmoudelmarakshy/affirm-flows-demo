<!--
Affirm USA — Virtual Card (VCN) integration reference
Source: "Affirm Integration Guide Generator" (QuickHost presentation 7a8aab1c),
config: Region USA + Virtual Card (VCN). Captured 2026-06-16 for the affirm-flows-demo accuracy pass.
This is generated reference content (authoritative endpoints/payloads), not hand-authored.
-->

Your Affirm Integration Guide
USA
Virtual Card (VCN)
Virtual Card (VCN)

Affirm issues a single-use virtual Visa card at checkout that your PSP charges like a standard card — captures, voids, and refunds all flow through your existing PSP, not Affirm's API.

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
Step 2 — Virtual Card: Open Affirm Modal

The frontend setup is identical to Direct API. The difference is in what your backend does after receiving the checkout_token.

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

affirm.checkout.open();
Step 3 — Virtual Card: Exchange Token for Card Credentials

When your user_confirmation_url receives the checkout_token, exchange it for virtual card credentials. Both environment VCN API bases are defined in the code below — swap to production before going live.

Python — VCN token exchange
Copy
import requests
from requests.auth import HTTPBasicAuth

# Select your environment — swap to production before going live
VCN_API_BASE = "https://sandbox.affirm.com/api/v2"        # Sandbox (testing)
# VCN_API_BASE = "https://api.affirm.com/api/v2"   # Production

def exchange_token_for_vcn(checkout_token):
    url = f"{VCN_API_BASE}/checkout/{checkout_token}/vcn"

    response = requests.post(
        url,
        auth=HTTPBasicAuth("YOUR_PUBLIC_KEY", "YOUR_PRIVATE_KEY"),
        headers={
            "Content-Type": "application/json",
        }
    )
    response.raise_for_status()
    vcn_data = response.json()

    # IMPORTANT: Store vcn_data["id"] for your records
    # NEVER log card_number, cvv, or expiration
    return vcn_data

# Response shape:
# {
#   "id": "XXXX-XXXX",          ← Store this
#   "card_number": "4111...",   ← PCI-sensitive
#   "cvv": "123",               ← PCI-sensitive
#   "expiration": "0228",       ← PCI-sensitive
#   "cardholder_name": "AFFIRM VIRTUAL",
#   "balance": 49900,
#   "billing": { ... }
# }
Python — Charge through PSP (Stripe example)
Copy
import stripe

def charge_virtual_card(vcn_data, amount_cents, order_id):
    # Create a payment method from the virtual card credentials
    payment_method = stripe.PaymentMethod.create(
        type="card",
        card={
            "number": vcn_data["card_number"],
            "exp_month": int(vcn_data["expiration"][:2]),
            "exp_year": int("20" + vcn_data["expiration"][2:]),
            "cvc": vcn_data["cvv"],
        },
        billing_details={
            "name": vcn_data["billing"]["name"]["first"] + " " + vcn_data["billing"]["name"]["last"],
            "address": {
                "line1": vcn_data["billing"]["address"]["line1"],
                "city":  vcn_data["billing"]["address"]["city"],
                "state": vcn_data["billing"]["address"]["state"],
                "postal_code": vcn_data["billing"]["address"]["zipcode"],
                "country": "US",
            },
        },
    )

    payment_intent = stripe.PaymentIntent.create(
        amount=amount_cents,
        currency="usd",
        payment_method=payment_method["id"],
        confirm=True,
        metadata={
            "order_id": order_id,
            "affirm_vcn_id": vcn_data["id"],
        },
    )
    return payment_intent
Python — Refund through PSP (Stripe example)
Copy
def refund_vcn_charge(payment_intent_id, amount_cents=None):
    """
    Refund a VCN charge through your PSP.
    amount_cents=None performs a full refund.
    VCN refunds go through your PSP — NOT Affirm's Transactions API.
    """
    params = {"payment_intent": payment_intent_id}
    if amount_cents is not None:
        params["amount"] = amount_cents  # Partial refund

    return stripe.Refund.create(**params)
Environment Reference

Quick reference for all URLs. Use sandbox for testing and development; production for live merchant traffic.

Sandbox & Production URLs
Copy
Region: USA

SANDBOX (testing):
  Affirm.js script:      https://cdn1-sandbox.affirm.com/js/v2/affirm.js
  VCN API base:          https://sandbox.affirm.com/api/v2

PRODUCTION (go-live):
  Affirm.js script:      https://cdn1.affirm.com/js/v2/affirm.js
  VCN API base:          https://api.affirm.com/api/v2

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
[ ] Set VCN_API_BASE to production URL in server-side code (see Step 3)
[ ] Update to live private API key (server-side only)
[ ] Switch PSP to production mode
[ ] Confirm PCI scope with your compliance team
[ ] End-to-end smoke test with a real transaction in production
[ ] Error tracking configured for Affirm interactions
[ ] Alerts configured for failure thresholds

Need a different configuration?

← Generate a new guide
