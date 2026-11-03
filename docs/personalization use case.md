# Abandoned-Cart WhatsApp Nudge — Design Notes

Use case. When a signed-in shopper adds products to a cart but does not purchase, send one personalized WhatsApp reminder after 30 minutes of inactivity. Suppress the nudge if the shopper purchases, clears the cart, opts out, or has already received a recent reminder.

## Chosen architecture and flow

The e-commerce service remains the system of record for carts. Its changes are captured through CDC and published to Kafka. Flink consumes cart events and maintains keyed state per customer/cart: every cart update resets a 30-minute inactivity timer. If the timer expires and no purchase/clear event has superseded it, Flink emits an AbandonedCart event. The recommendation engine checks the latest eligible customer profile/consent snapshot, reads that customer’s precomputed Top-K products from the Feature Store, selects a suitable item or message, and hands the WhatsApp template to the WhatsApp provider. Delivery and response events can be returned to Kafka for measurement.

## Options considered

Option A — Precomputed Top-K in the Feature Store (chosen): Airflow/CDP jobs refresh customer segments and features; the batch recommendation job uses product/customer features and Vector DB similarity to generate ranked Top-K lists. The real-time trigger performs a small key lookup. This gives predictable low latency, low per-message compute cost, and no repeated vector search or analytical reads against the operational DB.

Option B — Query the Vector DB on every abandoned-cart trigger: use the cart/customer context to retrieve similar products at send time. This can be fresher and more flexible, but adds network and retrieval latency, increases online compute/load and operational complexity, and requires careful filtering/ranking at trigger time. It is unnecessary for a reminder whose primary requirement is a fast, relevant suggestion.

## Over-engineering check and gaps

The design stays within the six high-level architecture groups. Flink keyed state provides the inactivity timer, so no separate scheduler or cache is introduced. The Vector DB is retained only for batch candidate generation; it is not required on the hot path. The operational DB is written by the normal cart flow but is not queried by the nudge path. One capability should be made explicit in implementation: a consent/eligibility snapshot and suppression rules (purchase, opt-out, frequency cap). These can be represented as CDP/profile data plus Flink/recommendation-engine logic rather than adding a new platform component. If the high-level diagram does not imply delivery-status ingestion, make the WhatsApp-to-Kafka feedback arrow explicit.

## Illustrative latency budget

After the 30-minute inactivity threshold: Flink detection/event emission 150 ms; eligibility and Feature Store lookup 200 ms; recommendation selection/message assembly 250 ms; WhatsApp API handoff 1,000 ms. Target P95 handoff is approximately 1.6 seconds, excluding the provider’s downstream delivery time. These are design budgets to validate with load testing, not measured guarantees.