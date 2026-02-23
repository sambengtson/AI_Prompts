# Module: Billing / Payments (Stripe)

Adds Stripe payment processing with webhook handling and entitlement tracking.

---

## Server Additions

### StripeController

- Route: `/api/stripe`
- `POST /checkout` — Create Stripe Checkout session (one-time or subscription)
- `POST /portal` — Create Stripe Customer Portal session for subscription management
- `GET /status` — Return current billing/entitlement status for the authenticated user
- `POST /webhook` `[AllowAnonymous]` — Handle Stripe webhook events

### Webhook Pattern

The webhook endpoint reads the raw body from the `X-Raw-Body` header (set by `LambdaEntryPoint.PostMarshallRequestFeature`) to preserve exact bytes for Stripe signature verification. API Gateway may re-encode the body, which breaks signature checks.

Key decisions:
- **Event types to handle**: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`
- **Entitlement model**: Track both purchase `Platform` (iOS/Android/Stripe) and `Type` (one-time purchase vs subscription) to support multiple payment sources
- **Checkout session vs Payment Intent**: Use Checkout Sessions for the simplest integration; Payment Intents for custom payment flows

### EntitlementService

Tracks what features/products users have access to based on Stripe events. Core operations:
- Grant entitlement on successful checkout/subscription
- Revoke/expire entitlement on subscription cancellation
- Extend expiration on invoice payment
- Query active entitlements for a user

### StripeSettings

Settings class with `FromEnvironmentAsync()` factory. Reads `STRIPE_SECRET_ARN` from environment, fetches API key and webhook secret from Secrets Manager.

### DynamoDB Key Pattern (if using DynamoDB)

| Entity | PK | SK |
|--------|----|----|
| Entitlement | `USER#{userId}` | `ENTITLEMENT#{productId}#{txnId}` |

### NuGet Package

```xml
<PackageReference Include="Stripe.net" />
```

---

## Infrastructure Additions

### SecretsStack

Add a Stripe secret containing API key and webhook secret:
```
{projectname}/stripe/{stage}
```

### ApiStack

- Pass Stripe secret ARN as Lambda environment variable: `STRIPE_SECRET_ARN`, `STRIPE_SECRET_NAME`
- Grant Secrets Manager read permission for the Stripe secret

---

## Web Additions (if Web client)

Optional pages:
- **Pricing.razor** — Display pricing tiers with checkout buttons
- **CheckoutSuccess.razor** — Post-checkout confirmation
- **CheckoutCancel.razor** — Checkout cancellation landing
- **Subscription.razor** — Subscription management (links to Stripe Customer Portal)

---

## Mobile Additions (if Mobile client)

- In-app purchase flows should redirect to a WebView or system browser for Stripe Checkout
- Entitlement status is queried via the API after purchase completion
- Consider Apple/Google in-app purchase requirements for App Store compliance — Stripe may only be usable for physical goods or services consumed outside the app
