<!--
Affirm — Travel vertical: itinerary object (+ insurance) integration reference
Source: "Affirm Integration Guide Generator" (QuickHost presentation 7a8aab1c),
config: Region USA + Direct Checkout + Vertical: Travel (Itinerary Object).
Captured 2026-06-16. Generated reference content (authoritative), not hand-authored.
-->

Your Affirm Integration Guide
USA
Direct Checkout
✈ Travel
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
Travel Vertical — Context & Guidance

Sections below are drawn from the Affirm Travel Technical Playbook.

Travel vertical overview
Copy
Affirm Travel Vertical covers: OTAs, Airlines, Lodging (Hotels + Vacation Rentals),
Cruises, Rental Cars, and Ticketing.

Key business drivers:
- Higher AOV and upsell opportunities (seat upgrades, room upgrades, excursions)
- Rapid YoY GMV growth; still an "untapped" market in many sub-verticals
- VCN is the dominant integration type due to multi-supplier fund distribution needs
Travel — Itinerary Object (Required)

The itinerary object is mandatory for all travel merchant checkouts. It provides fraud/identity underwriting signals.

Why it is required
Copy
The itinerary object passes travel-specific data to Affirm's underwriting model:
- Destination
- Travel start/end dates
- Other booking metadata

These signals improve fraud detection and identity verification.
Example: If fraud rings target a specific location, the itinerary destination
         allows Affirm to flag and further evaluate checkout requests.

Reference: https://docs.affirm.com/developers/reference/the-itinerary-object
JS — Itinerary object in checkout payload
Copy
affirm.checkout({
  merchant: { /* ... */ },
  shipping: { /* ... */ },
  items: [ /* ... */ ],
  order: { /* ... */ },

  // REQUIRED for travel merchants
  itinerary: {
    travel_type: "flight",          // "flight" | "hotel" | "car_rental" | "cruise" | "event"
    departure_time: "2026-06-15T08:00:00",
    arrival_time:   "2026-06-15T14:00:00",
    origin: {
      city: "New York",
      country: "US",
      airport_code: "JFK"
    },
    destination: {
      city: "Los Angeles",
      country: "US",
      airport_code: "LAX"
    },
    passengers: [
      { name: { first: "Jane", last: "Smith" }, ticket_number: "AA123456" }
    ]
  }
});
Travel — Insurance Add-On Support

Insurance integration requirements for your selected integration type. Full internal doc →

VCN Toggle Solution — Overview
Copy
Direct API merchants who offer optional insurance at checkout should use the
VCN Toggle Solution: dynamically switch between Direct API and VCN based on
whether insurance is in the cart.

  No insurance  →  Direct API  (use existing Direct API keys)
  Insurance      →  VCN         (use separate VCN keys; insurance provider
                                  charges their portion directly via the virtual card)

Two separate sets of Affirm API keys are required — one for each integration.
Coordinate with your Affirm TAM to provision both before go-live.
JS — Toggle Direct API ↔ VCN based on insurance
Copy
// Switch Affirm integration type based on whether insurance is in the cart
function openAffirmCheckout(cart) {
  const hasInsurance = cart.items.some(item => item.isInsurance);

  affirm.checkout(buildCheckoutObject(cart));

  if (hasInsurance) {
    // VCN path — insurance provider charges their portion via the virtual card
    affirm.checkout.open_vcn({
      checkout_complete: function(vcnData) {
        // 1. Charge booking portion through your PSP using vcnData card credentials
        // 2. Pass customer's REAL billing address (not vcnData.billing) to
        //    insurance provider alongside the card credentials
        handleVcnInsuranceCheckout(vcnData, cart);
      },
      checkout_cancel: function() { /* handle cancellation */ }
    });
  } else {
    // Direct API path — standard checkout, no insurance
    affirm.checkout.open();
  }
}
⚠ Billing Details — CRITICAL REQUIREMENT
Copy
Some insurance providers require the customer's real billing address to
authorize the transaction, complete fraud checks, and successfully issue the policy.

THE PROBLEM:
  Affirm VCN always returns Affirm's HQ address as the virtual card billing
  address — NOT the customer's address. Passing this to the insurance provider
  may cause the transaction to be declined or the policy to fail to issue.

BEFORE GOING LIVE — confirm with the insurance provider:
  Does your insurance provider require a valid customer billing address to
  authorize and issue the policy?

  YES → Merchant must collect the customer's billing address at checkout and
        pass it to the insurance provider SEPARATELY from the VCN card details.
        Do not rely on the billing address returned with the VCN.

  NO  → No additional action needed.
JSON — Itinerary object with trip_insurance add-on (HIGHLY RECOMMENDED)
Copy
{
  "type": "flight",
  "sku": "a9dladiak",
  "display_name": "MIA-DCA-2026-06-15T12:07",
  "flight_number": "UA110",
  "origin": "MIA",
  "destination": "DCA",
  "date_start": "2026-06-15T12:07",
  "date_end": "2026-06-15T15:21",
  "fare_type": "economy",
  "num_travelers": 2,
  "travelers": {
    "0": {
      "name": "Jane Smith",
      "dob": "1990-04-12",
      "frequent_flyer_number": "true",
      "tsa_pre_enrolled": "true",
      "passport_number_on_file": "false"
    }
  },
  "add-ons": {
    "trip_insurance": 5300
  }
}

// Note: trip_insurance value is in cents (smallest currency unit)
// The add-ons attribute is required when insurance is included in the checkout
JSON — Items object with separate insurance line item (HIGHLY RECOMMENDED)
Copy
"items": {
  "flight-mia-dca": {
    "display_name": "Flight MIA to DCA",
    "sku": "flight-ua110-mia-dca",
    "unit_price": 44700,
    "qty": 1,
    "item_url": "https://yoursite.com/booking/12345"
  },
  "trip-insurance": {
    "display_name": "Trip Insurance",
    "sku": "allianz-trip-insurance",
    "unit_price": 5300,
    "qty": 1
  }
}

// Why a separate insurance line item matters:
//  • Surfaces in Merchant Portal under Charge Details for visibility
//  • Enables reliable partial dispute management (insurance-only disputes)
//  • Extends reporting to separate insurance volume from booking volume
//  • Values in cents (smallest currency unit)

Need a different configuration?

← Generate a new guide
