# Recipe — Subscribe Customers via a Payment Link

**Goal:** let customers subscribe via a **payment link** (typically a Stripe Payment Link or invoice) rather than going through the in-app `Checkout` widget. Useful when:

- The customer isn't authenticated in your app yet.
- Sales is closing the deal and wants a clean URL to send.
- You're handling billing through invoices rather than credit-card collection.

## When the in-app Checkout is the wrong shape

- Self-serve, signed-in upgrade flow → use `Checkout` instead (`stigg-recipes/references/checkout.md`).
- B2B custom-priced deals → use **custom plans** + sales-led provisioning, not payment links.

Payment links sit between these — one-step purchase with an existing tokenized payment method, or invoice-driven payment.

## Steps

### 1. Model the plan (`stigg-pricing-modeling`)

The plan being subscribed to via the payment link must already exist in Stigg's catalog (free / paid / custom). Note: payment links work most naturally with **paid plans** that have a clear price.

### 2. Configure your billing provider's payment link

In Stripe (or equivalent):

- Create a payment link for the plan's price.
- Set the success URL to a route in your app that knows how to provision the Stigg subscription.

### 3. Provision in Stigg post-payment (`stigg-subscriptions/references/provisioning.md`)

When the success URL is hit (or the corresponding webhook fires), provision the Stigg subscription:

```ts
// On payment-link success webhook / redirect:
await stigg.provisionCustomer({ customerId, name, email, billingId: stripeCustomerId });
await stigg.provisionSubscription({
  customerId,
  planId: 'plan-pro',
  billingPeriod: 'MONTHLY',
  // do NOT charge again — Stripe already collected
});
```

The Stigg subscription references the Stripe customer / payment method; subsequent recurring charges flow through Stripe.

### 4. Render the customer portal (`stigg-widgets/references/customer-portal.md`)

Once provisioned, the customer can manage their subscription, see usage, and update payment via the standard customer portal.

## Gotchas at the seams

- **Don't double-charge.** Stripe collected payment via the link; don't pass payment-collection options to `provisionSubscription` that re-charge.
- **Customer must exist before provisionSubscription.** Provision the customer first (or use the upsert pattern). The Stripe customer ID should land on Stigg's `billingId` field.
- **Payment link succeeds but provisioning fails** — the customer paid, you owe them the subscription. Wire the success webhook to retry provisioning, alert ops, and have a manual recovery flow.
- **Webhook ordering.** Stripe fires multiple events (`checkout.session.completed`, `customer.subscription.created`, etc.). Decide which one is your trigger and don't double-provision.
- **Authentication state at the success URL.** If the customer wasn't signed in when they paid, you may need a session-creation step before they can use the product. Plan that flow.

## Verify in production

- New customer clicks the payment link → completes payment in Stripe.
- Success webhook fires → Stigg customer + subscription provisioned.
- Customer redirected to your app → Stigg customer portal renders the new plan.
- Recurring charges flow through Stripe; Stigg subscription stays active.
- Full recovery flow tested: kill the provisioning step, replay the webhook, confirm the subscription provisions correctly.
