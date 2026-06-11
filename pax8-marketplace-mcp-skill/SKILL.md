---
name: pax8-marketplace-mcp-skill
description: >
  Use when the user asks about their Pax8 marketplace data — companies, subscriptions,
  products, quotes, orders, invoices, or usage. Provides tool selection, UUID resolution
  patterns, and a recipe library for common partner workflows.
license: Apache-2.0
compatibility: Requires the Pax8 MCP Server (mcp.pax8.com). Connect via OAuth2.1 (preferred) or a legacy MCP token — see devx.pax8.com/docs/mcp-server for setup instructions.
metadata:
  author: pax8
  version: "1.0"
---

# Pax8 Marketplace MCP Skill

## What This Skill Does

This skill teaches you to use the Pax8 Marketplace MCP Server (`mcp.pax8.com`) to answer questions about a partner's marketplace data. The MCP exposes tools covering companies, subscriptions, products, quotes, orders, invoices, usage, and catalog search. Use it to look up subscription status, pull invoice details, research product pricing, monitor usage, and run common account management workflows — all from a conversational prompt.

For connection setup, see `references/connection-guide.md`.

## Glossary

| Term | Meaning |
|---|---|
| **Partner** | You — the MSP with a Pax8 account. All data returned by the MCP belongs to your account. |
| **Company** (or Client) | One of your end customers, managed under your partner account. |
| **Product** | A licensable item in the Pax8 catalog (e.g., Microsoft 365 Business Premium, Acronis Cyber Protect). |
| **Subscription** | An active (or historical) commitment to a product for a specific company, with quantity and billing term. |
| **Order** | A purchase transaction that created or modified one or more subscriptions. |
| **Invoice** | A monthly billing document summarizing charges across all companies and subscriptions. |
| **Quote** | A saved pricing proposal for one or more products, associated with a company. |
| **Usage** | Consumption data for metered products (Azure, AWS, etc.) — summarized or line-level. |
| **PSA** | Professional Services Automation — a platform used by MSPs to manage tickets, billing, and client records (e.g., ConnectWise, Autotask). |
| **Metered products** | Products billed in arrears based on actual consumption (e.g., Azure, AWS). Charges are calculated after the billing period closes and appear on the following invoice. |
| **Quantity-based products** | Products billed ahead of time based on a fixed seat or license count (e.g., Microsoft 365). |
| **Proration** | A partial-period charge applied when a subscription starts, ends, or changes mid-billing-cycle. Calculated based on the product's billing anniversary date. |
| **NCE 7-day rule** | Microsoft New Commerce Experience subscriptions can be cancelled within 7 days of creation for a full refund. After that, cancellation takes effect at the end of the term. |

---

## Key Enums

**Subscription status:** `Active` · `Cancelled` · `PendingManual` · `PendingAutomated` · `PendingCancel` · `WaitingForDetails` · `Trial` · `Converted` · `PendingActivation` · `Activated`

**Billing term:** `monthly` · `annual` · `two-year` · `three-year` · `one-time` · `trial` · `activation`

**Company status:** `active` · `inactive` · `deleted`

**Quote status:** `draft` · `pending` · `sent` · `accepted` · `declined` · `expired` · `closed`

**Invoice status:** `unpaid` · `paid` · `void` · `carried` · `nothing due`

---

## Decision Guide

| User says... | Start with |
|---|---|
| "Show me [company name]'s subscriptions" | `pax8-list-companies` (get UUID) -> `pax8-list-subscriptions` |
| "What's the status of [company]'s [product] subscription?" | `pax8-list-companies` -> `pax8-list-subscriptions` (filter by companyId + productId) |
| "How many seats does [company] have for [product]?" | `pax8-list-companies` -> `pax8-list-subscriptions` -> `pax8-get-subscription-by-uuid` |
| "Show me the latest invoice for [company]" | `pax8-list-companies` -> `pax8-list-invoices` (filter by companyId, sort by date) |
| "What did we bill in [month]?" | `pax8-list-invoices` (invoiceDateRangeStart/End for that month) |
| "What's the price for [product]?" | `pax8-lookup-product` (get UUID) -> `pax8-get-product-pricing-by-uuid` |
| "Do we have a quote for [company]?" | `pax8-list-quotes(search: "Company Name")` |
| "What are [company]'s Azure charges this month?" | `pax8-list-companies` -> `pax8-list-subscriptions` -> `pax8-get-usage-summary` |
| "Show me line-by-line Azure usage for [company]" | `pax8-list-companies` -> `pax8-list-subscriptions` -> `pax8-get-usage-summary` -> `pax8-get-detailed-usage-summary` |
| "What orders did we place for [company]?" | `pax8-list-companies` -> `pax8-list-orders` |
| "Find me a backup product" | `pax8-semantic-search-product` |
| "Look up SKU [X]" | `pax8-lookup-product` |
| "How do I set up MCP?" | `get-pax8-help-documents` or `references/connection-guide.md` |

---

## UUID Flow Patterns

Most MCP tools require a UUID, but users always provide names. Always resolve name -> UUID before calling detail tools.

**Name -> UUID resolution:**

```text
pax8-list-companies(company_name: "Acme Co") -> returns companyId
pax8-lookup-product(productName: "Microsoft 365 Business Premium") -> returns productId
pax8-list-quotes(search: "Acme Co") -> returns quoteId
pax8-list-invoices(companyId: ...) -> returns invoiceId
```

**Common chains:**

```text
Company detail:    list-companies -> get-company-by-uuid
Subscription list: list-companies -> list-subscriptions(companyId)
Invoice detail:    list-companies -> list-invoices(companyId) -> get-invoice-by-uuid
Usage summary:     list-companies -> list-subscriptions(companyId) -> get-usage-summary(subscriptionId)
Usage detail:      ... -> get-usage-summary -> get-detailed-usage-summary(usageSummaryId, usageDate)
Product pricing:   lookup-product -> get-product-pricing-by-uuid
```

---

## Pagination

All list tools support `page` (0-indexed) and `size`. Default to `size: 10` for exploratory queries. Use `size: 25-50` only when the user needs a comprehensive view.

If results are truncated, tell the user how many were returned and offer to fetch the next page.

---

## Core Patterns

### Pattern 1 — Name-to-UUID

Never assume a UUID from a name. Always call a list tool first:

```text
User: "Show me Acme Co's subscriptions"
1. pax8-list-companies(company_name: "Acme Co")         -> companyId: "abc-123"
2. pax8-list-subscriptions(companyId: "abc-123")         -> subscription list
```

If `pax8-list-companies` returns multiple matches, present the list and ask the user to confirm which company they mean before proceeding.

### Pattern 2 — Usage Three-Step

Usage data requires resolving the subscription first, then summary, then detail only if needed:

```text
User: "What's Acme Co's Azure usage?"
1. pax8-list-subscriptions(companyId: "abc-123")               -> find the cloud subscription -> subscriptionId
2. pax8-get-usage-summary(subscriptionId: "sub-uuid")          -> usage summaries (includes billing period in results)
3. pax8-get-detailed-usage-summary(usageSummaryId: "...", usageDate: "2026-05-07")
   -> Only call this if the user wants line-level breakdown; usage data is posted on the 7th — use the 7th of the billing month as usageDate
```

---

## Recipe Library

Each recipe maps a common partner workflow to the MCP tool chain that fulfils it.

**Recipe format:** Trigger -> Tool chain -> Notes

---

### 1b — Quote Status & Details Check

**Trigger:** User wants to see the status or contents of a quote for a company.

**Tool chain:**

1. `pax8-list-quotes(search: "Company Name")` -> find the relevant quote
2. `pax8-get-quote-by-uuid(quoteId: ...)` -> full line items and totals

**Example prompt:**
> "What's in the quote we sent Acme Co last week?"

**Notes:** Present quote line items as a table (product, quantity, unit price, total). Show expiry date prominently if the quote is near expiration.

---

### 2b — Product & Pricing Lookup

**Trigger:** User wants to know the price of a product before quoting or ordering.

**Tool chain:**

1. `pax8-lookup-product(productName: ...)` -> `productId`
2. `pax8-get-product-pricing-by-uuid(productId: ...)` -> pricing tiers

**Example prompt:**
> "What's our cost for Microsoft 365 Business Premium?"

**Notes:** Show all available billing terms (monthly, annual). Flag if pricing varies by quantity tier. Pass `companyId` to `pax8-get-product-pricing-by-uuid` if the user wants customer-specific pricing.

---

### 3b — Upsell Opportunity Finder

> **Marketplace Agent:** The [Pax8 Marketplace Agent](https://app.pax8.com) includes an `opportunity-explorer` tool that handles this workflow with a single call. If the Marketplace Agent is available in your environment, prefer it. The recipe below is the manual alternative using MCP tools directly.

**Trigger:** User wants to identify upsell opportunities across their book of business.

**Tool chain:**

1. `pax8-list-subscriptions(status: "Active")` -> all active subscriptions
2. Cross-reference: look for companies with Office 365 but no backup, or Microsoft 365 but no security add-on

**Example prompt:**
> "Which of my companies have Microsoft 365 but no backup solution?"

**Notes:** This requires iterating over subscriptions and grouping by company. Present findings as a table: company, current products, suggested add-on. Remind the user that purchasing must be done at app.pax8.com.

---

### 4b — QBR Prep Assistant

> **Marketplace Agent:** The [Pax8 Marketplace Agent](https://app.pax8.com) includes a dedicated QBR tool that orchestrates this workflow automatically. If the Marketplace Agent is available in your environment, prefer it. The recipe below is the manual alternative using MCP tools directly.

**Trigger:** User is preparing a Quarterly Business Review for a company.

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-subscriptions(companyId: ...)` -> active subscriptions
3. `pax8-list-invoices(companyId: ..., invoiceDateRangeStart: ..., invoiceDateRangeEnd: ...)` -> last 3 months of invoices
4. `pax8-list-subscriptions(companyId: ...)` -> find cloud subscriptions -> `subscriptionId`
5. `pax8-get-usage-summary(subscriptionId: ...)` -> usage by billing period (if applicable)

**Example prompt:**
> "Help me prep for Acme Co's QBR — I need a summary of their subscriptions and spend over the last quarter."

**Notes:** Summarize as: active products + seat counts, total spend per month, any subscriptions expiring in the next 90 days.

---

### 6b — Pre-Order Information Gathering

**Trigger:** User wants to know what's required before placing an order for a product.

**Tool chain:**

1. `pax8-lookup-product(productName: ...)` -> `productId`
2. `pax8-get-product-ordering-requirements(productId: ...)` -> required fields and commitment terms

**Example prompt:**
> "What do I need to have ready before ordering Acronis Cyber Protect for a new client?"

**Notes:** List all required fields clearly. Remind the user that orders must be placed at app.pax8.com.

---

### 7b — New Subscription Verification

**Trigger:** User wants to confirm a subscription was created successfully.

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-subscriptions(companyId: ..., status: "Active")` -> find the subscription
3. `pax8-get-subscription-by-uuid(uuid: ...)` -> confirm quantity, term, start date

**Example prompt:**
> "Can you confirm that Contoso's new Microsoft 365 Business Premium subscription went through?"

**Notes:** Show subscription ID, status, quantity, billing term, and start date. If status is `PendingManual` or `PendingAutomated`, the subscription is still being provisioned.

---

### 8b — License Availability Check

**Trigger:** User wants to know how many seats are available under an existing subscription.

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-subscriptions(companyId: ...)` -> find the subscription
3. `pax8-get-subscription-by-uuid(uuid: ...)` -> quantity and assigned seats

**Example prompt:**
> "How many Microsoft 365 licenses does Acme Co have and how many are in use?"

**Notes:** The MCP returns licensed quantity. Actual assignment counts come from the vendor portal (Microsoft Admin Center). Clarify this distinction if the user asks about "used" vs "available" seats.

---

### 9b — Post-Sale Investigation

**Trigger:** User needs to look into what happened after a sale — order status, subscription state, or invoice.

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-orders(companyId: ...)` -> recent orders
3. `pax8-get-order-by-uuid(uuid: ...)` -> order line items and status
4. `pax8-list-subscriptions(companyId: ...)` -> resulting subscriptions

**Example prompt:**
> "Something went wrong with Acme Co's order last week — can you tell me what happened?"

**Notes:** Review order status in the returned data. Surface the order date, product, quantity, and current subscription status.

---

### 10b — Manual Documentation Update

**Trigger:** User wants a data export or summary to update an external system (PSA, spreadsheet, CRM).

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-subscriptions(companyId: ...)` -> full subscription list
3. `pax8-get-subscription-by-uuid(uuid: ...)` for each -> detailed fields

**Example prompt:**
> "Give me a full list of Acme Co's subscriptions with product name, quantity, billing term, and renewal date so I can update ConnectWise."

**Notes:** Format output as a markdown table. The user will copy-paste into their PSA.

---

### 11b — Ad-Hoc Subscription Audit

**Trigger:** User wants a full view of subscriptions across their entire book of business, or for one company.

**Tool chain:**

- All companies: `pax8-list-subscriptions(status: "Active")` — paginate until all returned
- One company: `pax8-list-companies(company_name: ...)` -> `pax8-list-subscriptions(companyId: ...)`

**Example prompt:**
> "Give me a list of every active subscription across all my clients."

**Notes:** For large books of business, this may require many paginated calls. Consider asking the user to scope by company first. Group results by company and present as a summary table.

---

### 12b — Offboarding Investigation

**Trigger:** User is offboarding a client and wants to see all active subscriptions to cancel.

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-subscriptions(companyId: ..., status: "Active")` -> all active subscriptions

**Example prompt:**
> "Acme Co is leaving us. What subscriptions do they currently have?"

**Notes:** List all active subscriptions with product name, quantity, billing term, and renewal date. Remind the user about the NCE 7-day rule for any Microsoft subscriptions. Cancellations must be done at app.pax8.com.

---

### 15 — Mid-Month Azure/AWS Usage Monitoring

**Trigger:** User wants to check cloud consumption spend before the invoice closes.

**Tool chain:**

1. `pax8-list-companies(company_name: ...)` -> `companyId`
2. `pax8-list-subscriptions(companyId: ..., status: "Active")` -> find the cloud subscription -> `subscriptionId`
3. `pax8-get-usage-summary(subscriptionId: ...)` -> usage summaries; billing period is included in each result
4. `pax8-get-detailed-usage-summary(usageSummaryId: ..., usageDate: "YYYY-MM-DD")` -> line-level breakdown if needed

**Example prompt:**
> "How much has Acme Co spent on Azure so far this month?"

**Notes:** The usage summary tool requires a `subscriptionId` — resolve the cloud subscription first. The detailed tool requires a `usageSummaryId` (from the summary response) and a `usageDate` in `YYYY-MM-DD` format. Usage data for a completed billing period is posted on the 7th of the following month; use the 7th of the billing month as `usageDate` (e.g., `2026-05-07` for May 2026). The summary tool is sufficient for totals; use the detailed tool only if the user wants line-level breakdown.

---

### 16 — End-of-Month Invoice Reconciliation

**Trigger:** User wants to reconcile their Pax8 invoice against their PSA or expected charges.

**Tool chain:**

1. `pax8-list-invoices(invoiceDateRangeStart: "YYYY-MM-01", invoiceDateRangeEnd: "YYYY-MM-31")` -> invoices for the month
2. `pax8-get-invoice-by-uuid(uuid: ...)` -> line-level breakdown

**Example prompt:**
> "Pull last month's invoice and break it down by client."

**Notes:** Group line items by company. Flag any charges that are significantly higher or lower than the prior month if prior-month data is available in context.

---

### 17b — Interactive Invoice Analysis

**Trigger:** User wants to understand, question, or dispute specific charges on an invoice.

**Tool chain:**

1. `pax8-list-invoices(companyId: ..., invoiceDateRangeStart: ..., invoiceDateRangeEnd: ...)` -> find the invoice
2. `pax8-get-invoice-by-uuid(uuid: ...)` -> full line items
3. `pax8-get-subscription-by-uuid(uuid: ...)` -> for any line items that need clarification

**Example prompt:**
> "Acme Co's invoice seems higher than usual this month. Can you walk me through the charges?"

**Notes:** Present charges as a table sorted by amount descending. For large or unusual line items, pull the underlying subscription detail to explain the charge (quantity change, billing term, proration, etc.).

---

## Limitations & Scope

- **Creating quotes, placing orders, or modifying data** -> direct to [app.pax8.com](https://app.pax8.com) or the relevant workflow at [devx.pax8.com/recipes](https://devx.pax8.com/recipes)
- **Vendor-specific configuration** (Microsoft Admin Center settings, Azure portal, etc.) -> direct to the vendor portal
- **"How does Pax8 work?"** general platform questions -> use `get-pax8-help-documents` rather than the marketplace data tools
- **Real-time provisioning status** — subscription status reflects what Pax8 has recorded; actual provisioning happens at the vendor and may lag
- **Pricing guarantees** — pricing returned by `pax8-get-product-pricing-by-uuid` reflects catalog pricing at query time; confirm at app.pax8.com before committing to a customer quote

---

## Behavioral Notes

- **Subscription status** — `PendingManual` means the subscription is being provisioned manually; `PendingAutomated` means automated provisioning is in progress
- **Multiple company matches** — if `pax8-list-companies` returns more than one result, list them and ask the user to confirm which company they mean before proceeding
- **Pagination** — if results are capped at the page size, inform the user how many were returned and offer to fetch the next page
- **Actions beyond the available tools** — acknowledge what the user wants and direct them to [app.pax8.com](https://app.pax8.com)
