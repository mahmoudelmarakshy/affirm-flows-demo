# Sample prompts

Copy/paste and tweak. Once the skill is installed, you can also name it
explicitly: prefix with `/affirm-integration-demo`.

## Rebrand the cart & theme

> Rebrand the demo for **Casper**. Navy/charcoal theme. Cart: Original Mattress
> (Queen) $1,095 qty 1, Foam Pillow $65 qty 2. Add a "Launch 10% off" discount.
> Tax $92.40, free shipping. Then show me a screenshot.

> Make this demo for **Williams Sonoma** — warm red theme, a $389.00 stand mixer
> and a $59.00 utensil set, 8.5% tax. Update the search placeholder and nav.

## Spin up a brand-new demo

> Create a new Affirm demo for **H&R Block** in a `hrblock-demo/` folder from the
> template, set it up as a $499 tax-prep service order (no shipping), and deploy
> it to GitHub Pages.

> Clone the template into a new repo `acme-affirm-demo`, rebrand for **Acme
> Electronics** with a $2,400 cart, and give me the Pages URL.

## Change identifiers / sample data

> Change the merchant `order_id` to `WS-55012`, the Affirm transaction id to
> `9RG3-PMSE`, and the sandbox pin to `000111`.

> Swap the sample VCN card to number 4242 4242 4242 4242, exp 12/30, and update
> the cardholder name.

## Change flow behavior

> Switch the demo's default flow to **VCN** and make the cart a single $1,899
> appliance with 9.25% tax.

> In the VCN flow, change the payment processor name in the log from
> "processor.com" to "Adyen".

## Extend the flow

> Add a step to the **Direct API** flow, right after Capture, showing the
> merchant receiving an `order.captured` webhook from Affirm. Link the Affirm
> docs on it.

> Add an `onValidationError` branch to the Direct API modal that shows what the
> log looks like when address validation fails.

## Preview & ship

> Screenshot the demo with the **VCN** flow run to completion.

> Deploy the current demo to GitHub Pages and confirm the live URL serves the
> latest version.
