# Business Requirements Document: MercatorPay

## Executive Summary
Marketplace platform for independent mapmakers to sell digital maps. Buyers browse, purchase, and download; sellers upload, price, and get paid out monthly. Must handle EU + US tax compliance from day one.

## Stakeholders
| Role | Name / Group | Concern |
|------|--------------|---------|
| Buyer | Hobbyist & professional GIS users | Easy discovery, secure download |
| Seller | Independent cartographers | Get paid reliably, low fees |
| Platform Admin | MercatorPay ops | Moderation, payouts, dispute handling |
| Tax Authority | EU VAT, US state sales tax | Correct collection and remittance |

## Goals & Success Metrics
- 500 active sellers within 6 months.
- 90% of payouts completed within 2 business days of monthly cycle.
- Zero tax filing exceptions in first audit.

## Scope
### In Scope
- Seller onboarding (KYC), map upload, pricing, payouts (Stripe Connect).
- Buyer search, browse, checkout, download with signed URLs.
- Admin moderation queue and dispute resolution workflow.
- EU VAT (OSS) and US sales tax (per state) collection at checkout.
### Out of Scope
- Physical map fulfillment.
- Affiliate program.
- Native mobile apps (web responsive only).

## Functional Requirements
### FR-1: Seller onboarding & KYC
**Description:** Seller signs up, provides identity + payout info, passes Stripe Connect KYC before listing maps.
**Acceptance Criteria:**
- Seller cannot publish a map until KYC status = approved.
- Failed KYC shows reason and remediation steps.
**Priority:** Must

### FR-2: Map upload and metadata
**Description:** Seller uploads map file (PDF, GeoTIFF, MBTiles), sets title, description, price, license, preview image.
**Acceptance Criteria:**
- File size up to 2 GB supported via chunked upload with resume.
- Required metadata enforced before publish.
**Priority:** Must

### FR-3: Buyer search & browse
**Description:** Buyer can search by keyword, filter by region/scale/format, and view map detail page.
**Acceptance Criteria:**
- Search returns results in <500 ms p95.
- Filters combinable.
**Priority:** Must

### FR-4: Checkout with tax
**Description:** Buyer purchases one or more maps; tax is computed correctly for buyer location.
**Acceptance Criteria:**
- EU buyers see VAT inclusive total per OSS rules.
- US buyers see state sales tax for taxable states.
- Receipt PDF emailed with tax breakdown.
**Priority:** Must

### FR-5: Secure download
**Description:** After purchase, buyer downloads map via short-lived signed URL.
**Acceptance Criteria:**
- URL expires after 24 hours; up to 3 re-issues from order page.
- Download is logged per purchase.
**Priority:** Must

### FR-6: Seller payout
**Description:** Monthly payout to seller's Stripe Connect account net of platform fee.
**Acceptance Criteria:**
- Payout runs on 1st of month covering prior month sales.
- Seller can see ledger and download CSV.
**Priority:** Must

### FR-7: Admin moderation
**Description:** Admin reviews flagged maps and decides to keep, remove, or warn seller.
**Acceptance Criteria:**
- Flagged maps appear in queue with reason.
- Decision is logged with admin user and rationale.
**Priority:** Should

### FR-8: Dispute resolution
**Description:** Buyer can open a dispute on an order; admin mediates.
**Acceptance Criteria:**
- Dispute states: open, awaiting-seller, awaiting-buyer, resolved-refund, resolved-no-action.
- Resolution triggers Stripe refund where applicable.
**Priority:** Should

### FR-9: Seller analytics
**Description:** Seller sees sales, views, conversion rate.
**Priority:** Could

## Non-Functional Requirements
### NFR-1: Performance
Search p95 <500 ms; checkout p95 <1.5 s.

### NFR-2: Compliance
GDPR-compliant data handling for EU users; tax reports exportable for OSS and Avalara.

### NFR-3: Availability
99.9% uptime for buyer-facing flows; 99.5% for admin.

## Constraints
- Must use Stripe Connect for payouts (already contracted).
- Must integrate with Avalara for US sales tax calculation.

## Assumptions
- All sellers have a bank account supported by Stripe Connect.
- Buyers pay by card (no crypto, no ACH at launch).

## Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Stripe KYC rejection rate too high | Medium | High | Pre-launch test with 20 sellers |
| Tax miscalculation | Low | High | Avalara + manual reconciliation first 3 months |
