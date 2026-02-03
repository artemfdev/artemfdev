# Real-Time at the Edge: Unified Tag-Based Caching with Fastly + Redis (Rails)

## Executive summary

I rebuilt a revenue-critical Rails site around **surrogate-key (tag) caching** so pages and APIs update in real time without hammering origin. I started with **Fastly tag purging on the offers API**, then extended the **same tag semantics** to **site-wide pages**, where I introduced **long-lived `Surrogate-Control` with `stale-while-revalidate` and `stale-if-error`**. Finally, I added a **Redis tag-aware page cache** that shares the same purge pipeline and uses **rebuild-on-purge**. Result: **CDN-speed delivery** with **database-level correctness**, stable p95 ≈ **87ms**, **zero timeouts**, and a **smaller infra bill**.

**At-a-glance**

- **Server response time, p95:** 1.7s / 30s (under crawlers) ➜ **87ms** (under crawlers)
- **Reliability:** crawler-driven timeouts ➜ **zero timeouts**
- **Infra footprint:** 6 dynos ➜ **2 dynos**, autoscaler retired
- **Throughput:** **4.5k** pages rebuilt in **11 min** (10 Sidekiq dynos)
- **Cost:** **$20/mo** for **1 GB** Redis (**60-70 KB/page** compressed)
- **Correctness & discipline:** **305 tests**, rebuild-on-purge + daily full rebuild

---

## Context & stakes

The site’s business model is **sponsored offers (PPC)**, so latency directly affects conversion and revenue. The **offers API** returned about **600 ms** in lab tests and exceeded **>1 s** under campaign load. **Separately**, heavy **crawler** activity on **general pages** overloaded origin, with **site p95 spiking to 30 s at 0.6 rps** on **6 dynos**, causing timeouts. I needed to keep data **fresh** (limits/eligibility change on clicks) while cutting **server work** to the bone.

---

## What I built (one purge, two caches)

**Design principle:** express freshness in **tags** (= model instances), not URLs—then make **both** the CDN and origin cache **honor those tags**.

- **Tags at the data model.** Each model gets a scoped prefix; e.g., a `Company` with id 100 yields **`cp/100`**.
- **Tags in responses.** Controllers/views push used tags via `include_surrogate_keys_for`. Global before_action `set_default_expires` also sets long-lived **`Surrogate-Control`** with **`stale-while-revalidate` (1 day)** and **`stale-if-error` (1 week)**.
- **Model-driven purges.** Model changes (writes/state transitions and click-driven limits) trigger asynchronous purges via model hooks for real-time freshness.
- **Unified purge pipeline.** A single Sidekiq job **`PurgeByKeyJob`** purges **Fastly** and **updates Redis** using the same tags.
- **Rebuild on purge.** Instead of “delete and hope for a miss,” I proactively **rebuild** affected pages after every purge to keep caches warm during releases.

This rolled out in three focused stages.

---

## Stage 1 — Test waters with Fastly surrogate-key caching for the API

I started with the mission-critical offers API:

- **Surrogate-Keys** emit model tags (e.g., `cp/100`) with every API response using `include_surrogate_keys_for` helper and global after_action `set_surrogate_key_headers`.
- **Asynchronous purges** via `PurgeByKeyJob` fire on writes and **click-driven** state changes by conditional hooks and delegation on the model.
- **Result:** **edge HITs 100–200 ms** with stable behavior during campaigns.

This removed load-related variability on the API without compromising freshness.

---

## Stage 2 — Apply tag semantics site-wide for pages

Next I moved the same **tag semantics** to user-facing pages rendered by Rails (embedding WordPress content):

- **Long-lived** `Surrogate-Control` **plus** `stale-while-revalidate` **(1 day)** and `stale-if-error` **(1 week)**.
- **Param allow-list + canonicalization** at the CDN to block cache-key explosion from PPC URLs.
- **Editorial workflow:** a lightweight **WordPress plugin** calls back into Rails so **page edits purge by tag**.

This offloaded most reads to the edge and flattened origin latency. The remaining pain surfaced when **crawlers** blew through long-tail pages that don’t stay resident at the edge.

---

## Stage 3 — Redis tag-aware page cache (same tags, origin-side)

To eliminate crawler-driven origin spikes on pages, I implemented a **Redis page cache** that uses **the same tag graph** as Fastly:

**Data model**

- `urls:/path` → page HTML (compressed with **Readthis** when >1 KB)
- `url_tags:/path` → set of tags present on that page
- `tags:tag` → set of pages that include that tag


**Behavior**

- **Writes** pipeline both set updates so a page and its tag links are updated atomically.
- **Purges** come from **`PurgeByKeyJob`**: update Fastly **and** Redis; then **rebuild on purge** to repopulate.
- **Release safety:** a **daily full rebuild** catches any missed invalidations and prunes junk keys.
- **Scale:** on releases and daily rebuilds **Sidekiq is autoscaled** to **10 dynos**; beyond that the database becomes the bottleneck, so I cap there.


**Enabling cache for a page** takes a single `before_action :cache_page`, which let me roll this out incrementally; the full site shipped in about two weeks.

---

## Observability & operations

I built **ActiveAdmin** views to operate and validate the system:

- **URL & Tag explorers:** preview rendered HTML (raw html/full page), see attached tags.
- **Purge history (last 1,000):** audit trail for CDN + Redis purges and WordPress-initiated edits.
- **WP purge verification:** quick checks to ensure editorial changes trigger the right tags.

This made rollouts and incident triage predictable and low-risk. (see Appendex for screenshots)

---

## Results

**Before**

- **Normal traffic:** p95 **1.7 s @ 0.2 rps**
- **Crawler surge:** p95 **30 s @ 0.6 rps**, frequent timeouts on **6 dynos**

**After**

- p95 **87 ms** at **0.7 rps** and **up to 2.0 rps**, **zero timeouts** (see Appendix for Heroku metrics screenshots)
- **2 dynos**, autoscaler no longer required
- Full rebuild **4.5k pages** in **11 min** at **10 Sidekiq dynos**
- **$20/mo** for **1 GB** Redis, thanks to compression (**60-70 KB/page** vs **250–300 KB raw**)


---

## Why it worked

- **Correctness via modeling, not heuristics.** I expressed freshness as a **page↔tag graph** (`urls:/path`, `url_tags:/path`, `tags:tag`), where tag = model instance. Invalidation is therefore **by meaning**, not by guessing URLs.
- **One purge, two caches.** The **same job** updates **edge** and **origin**. No split brain between CDN and Redis.
- **Real-time without chaos.** `Surrogate-Control` + `stale-*` keep the edge snappy; **rebuild-on-purge** keeps the origin cache warm before traffic hits it.
- **Ops safety nets.** Daily rebuild, soft purges on deploys, and admin tooling keep incidents boring.
- **Cost discipline.** Compression and tag scoping fit the whole site in **1 GB** of Redis and let me shut off the autoscaler.


---

## Trade-offs & how I handled them

- **Bulk tag deletion atomicity.** Very large tags require **batched** delete operations to stay safe and fast, but batches mean operation is not fully atomic. A future improvement is a small **Lua** script to encapsulate the sequence server-side; today’s rebuild-on-purge keeps partial progress safe.
- **Route/param hygiene.** I tightened route constraints on Rails and **allow-listed/sorted** query params on Fastly to prevent key explosions and junk variants.
- **APIs vs pages.** Pages benefit from rebuild-on-purge; for ultra-fresh API endpoints I can flip specific purges to **delete-then-miss** to force an immediate refetch.
- **Very frequent updates**. As with any cache, this pattern is not suitable for very frequent data updates, but can work well for data that is updated less frequently.

---

## What I’d bring to your team

- A **repeatable pattern** for **real-time, tag-based invalidation** that spans CDN and origin—cleanly integrated into Rails models/controllers and editorial tooling.
- **Measurable wins**: sub-100ms p95 at realistic rps, fewer servers, predictable releases, and a tiny cache bill.
- **Engineering maturity**: 300+ tests, safe rollouts, and admin views so the system can be run—not just admired.

If you need pages and APIs to feel **instant** without paying for a fleet, I’ve already built that playbook—and the numbers above are why it sticks.

---

## Appendix

### Heroku Metrics

#### Before Redis cache

![Before, 1.7s at 0.2 rps](https://i.imgur.com/gylkyPX.jpeg)

![Before 30.0s at 0.6 rps](https://i.imgur.com/Scp4WiL.jpeg)

#### After Redis cache

![After 0.6 rps](https://i.imgur.com/HVlrowR.jpeg)

![After 2.0 rps](https://i.imgur.com/Do64qr4.jpeg)

### ActiveAdmin

![Index of cached pages](https://i.imgur.com/1SPFvUi.jpeg)

![Index of cached tags](https://i.imgur.com/Pn9mqBB.jpeg)

![Cached page](https://i.imgur.com/GDoAxy9.jpeg)

![Purge history](https://i.imgur.com/s25rqFq.jpeg)

### Redis Cloud

![Metrics](https://i.imgur.com/zpAHMc7.jpeg)