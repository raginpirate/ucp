---
title: Shop Pay payment handler specification
description: >-
  Technical specification for integrating Shop Pay as a UCP payment handler for
  merchant stores and agents.
source_url:
  html: 'https://shopify.dev/docs/agents/checkout/shop-pay-handler'
  md: 'https://shopify.dev/docs/agents/checkout/shop-pay-handler.md'
---

# Shop Pay payment handler specification

The `dev.shopify.shop_pay` handler enables merchants to offer Shop Pay as an accelerated checkout option through UCP-compatible agents. [Shop Pay](https://www.shopify.com/shop-pay) is Shopify's native payment solution that provides a streamlined checkout experience by securely storing buyer payment and shipping information.

This handler enables a delegated payment flow where agents can securely generate a Shop Token for a buyer's selected payment instrument in Shop Pay. Merchants can then process these tokens, and agents can leverage the same Shop Pay integration across all participating merchants.

***

## Key benefits

* **Accelerated checkout:** Buyers with Shop Pay accounts can complete purchases faster using their saved payment and shipping details.
* **Delegated payments:** Agents can orchestrate Shop Pay payments without directly handling sensitive payment credentials.
* **Standardized integration:** A single Shop Pay integration works across all UCP-compatible merchants.

***

## Merchant integration

Merchants enable Shop Pay through UCP by advertising the handler in their payment configuration and processing Shop Pay tokens when buyers complete checkout.

### Requirements

Requirements depend on whether you're a Shopify merchant or integrating as an external merchant:

If you're a Shopify merchant, then Shopify automatically advertises all UCP payment handlers on your behalf, including Shop Pay.

If you're an external merchant, then you must register for Shop Pay to obtain your `shop_id` before advertising Shop Pay through UCP.

### Handler configuration

Merchants advertise Shop Pay support by including the handler in their payment handlers array. The Shop Pay handler uses a unified schema that defines three configuration levels: business, platform, and response.

#### Configuration levels

The handler schema defines distinct configuration contexts:

| Level | Context | Required Fields | Description |
| - | - | - | - |
| `business_schema` | Merchant discovery | `shop_id` | Merchants advertise their Shop Pay identifier for platforms to discover. |
| `platform_schema` | Platform discovery | `client_id` | Platforms advertise their registered agent identifier. |
| `response_schema` | Checkout response | `shop_id` | Runtime configuration returned during active checkout sessions. |

#### Business configuration (merchant)

Merchants declare Shop Pay in their payment handlers to enable platform discovery:

```json
{
  "ucp": {
    "payment_handlers": {
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay",
          "version": "2026-01-16",
          "spec": "https://shopify.dev/docs/agents/checkout/shop-pay-handler",
          "schema": "https://shopify.dev/ucp/shop-pay-handler/2026-01-16/schema.json",
          "config": {
            "shop_id": "shopify-559128571"
          }
        }
      ]
    }
  }
}
```

#### Platform configuration (agent)

Platforms declare their Shop Pay client credentials to enable merchant discovery:

```json
{
  "ucp": {
    "payment_handlers": {
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay",
          "version": "2026-01-16",
          "spec": "https://shopify.dev/docs/agents/checkout/shop-pay-handler",
          "schema": "https://shopify.dev/ucp/shop-pay-handler/2026-01-16/schema.json",
          "config": {
            "client_id": "agent-1234567890"
          }
        }
      ]
    }
  }
}
```

#### Response configuration (runtime)

During active checkout sessions, merchants return the response configuration:

```json
{
  "ucp": {
    "payment_handlers": {
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay",
          "version": "2026-01-16",
          "config": {
            "shop_id": "shopify-559128571"
          }
        }
      ]
    }
  }
}
```

### Payment instrument support

By advertising the Shop Pay handler, merchants indicate they can accept and process Shop Pay payment instruments. Merchants must be able to handle payment objects conforming to the Shop Pay instrument schema.

#### Instrument schema

Shop Pay instruments extend the base UCP payment instrument with Shop Pay-specific fields and credential types.

| Field | Type | Description |
| - | - | - |
| `id` required | String | Unique identifier for this payment instrument. |
| `handler_id` required | String | Must match the handler's `id` (for example, `shop_pay`). |
| `type` required | String | Must be `shop_pay`. |
| `credential` required | Object | The Shop Pay credential containing the token. |
| `credential.type` required | String | Must be `shop_token`. |
| `credential.token` required | String | The Shop Token from the delegated payment flow. |
| `billing_address` required | Object | Billing address associated with the Shop Pay account. |
| `display` | Object | Display information for this Shop Pay instrument. |
| `display.email` | String | The buyer's email address associated with the Shop Pay account. |

Merchants receive a payment object structured as follows when an agent submits a Shop Pay payment:

```json
{
  "payment": {
    "instruments": [
      {
        "id": "instr_shop_pay_1",
        "handler_id": "shop_pay",
        "type": "shop_pay",
        "credential": {
          "type": "shop_token",
          "token": "shop_abc123xyz789..."
        },
        "display": {
          "email": "buyer@example.com"
        },
        "billing_address": {
          "full_name": "Jane Doe",
          "street_address": "123 Main St",
          "address_locality": "San Francisco",
          "address_region": "CA",
          "postal_code": "94102",
          "address_country": "US"
        }
      }
    ],
    "selected_instrument_id": "instr_shop_pay_1"
  }
}
```

***

## Agent integration

Agents orchestrate Shop Pay payments on behalf of merchants through a delegated flow that collects payment authorization from buyers and returns a token for processing.

### Requirements

Before handling delegated Shop Pay payments, agents must register with Shop Pay to obtain a `client_id` that supports delegated experiences.

### Delegated payment protocol

Agents must follow this flow to process a `dev.shopify.shop_pay` handler:

#### Step 1: Discover handler

Identify `dev.shopify.shop_pay` in the merchant's `payment_handlers` map from the checkout response or merchant profile. The handler appears with the response configuration:

```json
{
  "ucp": {
    "payment_handlers": {
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay",
          "version": "2026-01-16",
          "config": {
            "shop_id": "shopify-559128571"
          }
        }
      ]
    }
  }
}
```

Agents can fetch the full handler schema from the merchant's business profile or platform discovery to access the schema URL.

#### Step 2: Build payment request

Build a Shop Pay payment request by:

1. **Initializing the delegated payment context:**

   * Provide the merchant's `shop_id` from the handler configuration.
   * Provide your agent's `client_id` obtained during registration.
   * This establishes that you're orchestrating a Shop Pay payment on behalf of the merchant.

2. **Constructing the payment request with standardized UCP checkout data:**

   * **Totals:** Subtotal, shipping, tax, and grand total amounts.
   * **Fulfillment options:** Available shipping methods or pickup locations.
   * **Line items:** Product details, quantities, and pricing.
   * **Currency and locale:** For proper formatting and display.

Present this payment request to the buyer through the Shop Pay interface, where they can authenticate and select their preferred payment method.

#### Step 3: Complete checkout

After the buyer confirms, Shop Pay returns a Shop Token. Wrap this token in a UCP payment instrument conforming to the Shop Pay instrument schema and submit the complete checkout request to the merchant:

```json
POST /checkout-sessions/{checkout_id}/complete
Content-Type: application/json
UCP-Agent: profile="https://agent.example/profile"


{
  "payment_data": {
    "id": "instr_shop_pay_1",
    "handler_id": "shop_pay",
    "type": "shop_pay",
    "credential": {
      "type": "shop_token",
      "token": "shop_abc123xyz789..."
    },
    "display": {
      "email": "buyer@example.com"
    },
    "billing_address": {
      "full_name": "Jane Doe",
      "street_address": "123 Main St",
      "address_locality": "San Francisco",
      "address_region": "CA",
      "postal_code": "94102",
      "address_country": "US"
    }
  }
}
```

Upon successful processing, the merchant returns the completed checkout state with an `order_id` confirming the purchase.

***

## Merchant processing

Upon receiving a `shop_pay` payment instrument, merchants must:

1. **Validate handler:** Confirm `handler_id` matches a configured Shop Pay handler.

2. **Extract token:** Retrieve the `token` from `credential.token`.

3. **Process payment:** Use the Shop Token to complete the payment through Shop Pay's payment processing API.

4. **Return response:** Respond with the finalized checkout state including order confirmation details.

***

## Security considerations

The Shop Pay handler implements multiple security measures to protect payment data and ensure proper authorization throughout the delegated payment flow.

### Token security

* **Single-use tokens:** Shop Tokens are designed to be single-use and can't be reused across transactions.
* **Time-limited:** Tokens have a limited validity period and should be used promptly after generation.
* **Checkout-scoped:** Tokens have context about the checkout and merchant, preventing usage outside of the verified checkout.
* **Secure transmission:** All token exchanges must occur over TLS 1.2+.

### Agent authorization

* Agents must register with Shop Pay to obtain a valid `client_id` before processing delegated payments.
* The `client_id` identifies the agent to Shop Pay and enables proper authorization tracking.

***

## Schema reference

The Shop Pay handler uses a unified schema that defines all configuration levels and payment instrument types:

* [Handler schema](https://shopify.dev/ucp/shop-pay-handler/2026-01-16/schema.json): The root schema defining all Shop Pay handler components including business, platform, and response configurations.

The schema references the following type definitions:

* [Business config](https://shopify.dev/ucp/shop-pay-handler/2026-01-16/types/business_config.json): Merchant-level configuration with `shop_id` for discovery.
* [Platform config](https://shopify.dev/ucp/shop-pay-handler/2026-01-16/types/platform_config.json): Platform-level configuration with `client_id` for agent registration.
* [Response config](https://shopify.dev/ucp/shop-pay-handler/2026-01-16/types/response_config.json): Runtime configuration returned in checkout responses.
* [Instrument schema](https://shopify.dev/ucp/shop-pay-handler/2026-01-16/types/shop_pay_instrument.json): Payment instrument structure extending the base UCP instrument.
* [Credential schema](https://shopify.dev/ucp/shop-pay-handler/2026-01-16/types/shop_pay_credential.json): Shop Token credential format for delegated payments.

***