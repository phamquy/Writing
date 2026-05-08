---
description: This one is a bit non technical, but very interesting read into ads system.
icon: adversal
---

# Programmatic Advertising Reference

### Ecosystem Overview

* **Advertisers/Agencies** set campaign goals, budgets, creatives, and tracking pixels via a DSP UI or advertiser ad server.
* **Demand-Side Platforms (DSPs)** translate marketer intent into config, evaluate bid requests in <120 ms, pick the best campaign, and submit a single bid per impression.
* **Ad Exchanges** operate the neutral auction fabric, using OpenRTB to fan out bid requests from many SSPs to many DSPs and determine winners.
* **Supply-Side Platforms (SSPs)** and **Ad Networks** aggregate publisher inventory, apply floors/yield rules, and forward impressions into exchanges.
* **Publishers & Publisher Ad Servers** control on-page slots, trigger auction calls, and render the winning creative for end users.
* **Users/Browsers/Apps** load the publisher page/app; sandboxed ad iframes render creatives and fetch assets (images, JS, VAST video) directly from advertiser CDNs.

### Communication & Protocols

* **OpenRTB** (HTTP/JSON, increasingly gRPC) standardizes bid requests/responses among SSPs, exchanges, and DSPs.
* Bid requests contain `imp`, `site/app`, `user`, `device`, `regs`, and `ext` objects covering placement specs, context, consent flags, identity tokens, and pricing floors.
* Bid responses carry a single bid per DSP seat with price, creative markup (`adm`), asset references, and tracking macros (e.g., `nurl`, `lurl`). Exchanges issue win notices back through the SSP.
* Event telemetry (impression, click, conversion) flows via tracking pixels/postbacks from publisher pages or advertiser sites to DSPs and advertiser ad servers for measurement.

### Data & Targeting Signals

* **User IDs / Cookies:** SSPs sync IDs with DSPs; the request’s `user.buyeruid` or `ext.eids` lets the DSP link to its audience profile.
* **Contextual Data:** `site`/`app` metadata (domain, app bundle, categories, keywords) plus brand-safety ratings inform contextual targeting when user IDs are sparse.
* **Device/Geo:** IP-derived geo, connection type, OS/browser from the `device` object fuel geo/device rules and compliance (e.g., COPPA, privacy regions).
* **Behavioral/First-Party Data:** Advertiser pixels/SDKs collect off-site interactions (product views, cart events). Those events populate segments in the DSP’s audience store; the raw events are not sent in bid requests—only the DSP’s derived segment flags are used during bidding.

### Auction & Bidding Mechanics

1. **Eligibility Filtering:** DSP indexes campaigns/line items by targeting rules (geo, device, inventory lists, consent, frequency caps) to shortlist candidates.
2. **Value Estimation:** ML models (tree ensembles, neural nets) predict expected CPA/ROAS/CTR per candidate impression. Pacing tokens and spend state remove over-budget lines.
3. **Bid Strategy:** Bid shading adjusts price expectations for first-price auctions. Strategy multipliers convert predicted value to a bid, respecting floors/ceilings and daily budgets.
4. **Creative Selection:** Multi-armed bandits or weighted rotations pick the best creative that meets slot specs. The DSP embeds that creative’s markup or advertiser ad-server tag in the response.
5. **Single Bid Submission:** Exchanges allow only one bid per DSP seat per impression to keep auctions fair; internal fairness (share-of-voice between campaigns) is enforced before the bid is emitted.
6. **Winner Determination:** The exchange/SSP compares all DSP bids, applies auction rules, and notifies the winner; publisher ad server renders the creative in the designated slot.

### Asset Delivery & Security

* The bid response usually includes HTML/JS or VAST markup referencing assets on advertiser CDNs; browsers fetch those assets directly. Standard iframe isolation prevents ads from snooping on the surrounding page, so off-slot interactions remain hidden from ad scripts.
* Publishers can expose limited signals via friendly iframes or SafeFrame APIs (e.g., viewability), but cross-site scripting defenses remain intact. Signed URLs or short-lived tokens are used only when advertisers restrict asset access (rare).

### Privacy, Compliance, and Safety

* Consent strings (IAB TCF) and regional flags (GDPR/CCPA/CPRA) travel in OpenRTB `regs`/`user.ext` fields; DSPs must check them before using IDs for personalized bidding.
* SSPs/DSPs enforce blocklists, brand-safety categories, and fraud detection (invalid traffic, bot patterns) both pre-bid and post-bid.
* Audit logs and immutable storage capture sensitive actions (manual overrides, consent handling) for compliance reviews.

### Measurement & KPIs

* **CPA (Cost Per Acquisition):** $CPA = \frac{\text{Spend\}}{\text{Conversions\}}$.
* **ROAS (Return On Ad Spend):** $ROAS = \frac{\text{Revenue\}}{\text{Ad Spend\}}$.
* Additional metrics: CTR, CVR, viewability, VCR, pacing status, incremental lift. Attribution systems align conversions to impressions/clicks via lookback windows.

### Alignment & Trade-Offs

* DSPs optimize advertiser outcomes (lower CPA, higher ROAS) while SSPs maximize publisher yield. Open auctions expose mismatches: if inventory is overpriced, DSP bids drop; if DSPs underpay, SSPs route impressions elsewhere. Supply-path optimization (SPO) helps advertisers favor transparent SSP/exchange routes.
* Fairness across campaigns is handled inside the DSP with pacing, rotation weights, and quotas—not by sending multiple bids per auction.

### Key Takeaways

* The most critical engineering surface is the **real-time bidding loop**, where latency, accuracy, and compliance intersect.
* **OpenRTB** is the lingua franca connecting independent SSPs, exchanges, and DSPs; extensions carry consent, identity, and optimization hints.
* **User data** from advertiser pixels powers targeting but stays within the DSP; bid requests simply include IDs so the DSP can apply its own segments.
* **Publisher ad servers** ultimately render the winning creative; even though DSPs supply markup, asset delivery happens client-side with standard iframe sandboxing.
* Robust **privacy, fraud, and observability** layers are mandatory to sustain trust and meet regulatory requirements while scaling auctions to millions of QPS.
