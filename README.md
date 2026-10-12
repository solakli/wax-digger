# Wax Digger

See what from your Discogs wantlist is for sale, and buy it for the cheapest total price
including shipping. Live at [waxdigger.ai](https://waxdigger.ai).

This is a write-up. The source is private. UI design and the May 2026 rebrand are by Dan.

## The problem

Buying records online is a search problem with a nasty cost function. A record you want
might be in stock at seven different places at seven different prices, and every store
charges shipping once per order. Buying five records from five stores costs five shipping
fees. The cheapest individual prices almost never produce the cheapest total.

Nobody solves this by hand across dozens of sellers and a 200-item wantlist, so people
overpay or give up and buy from one shop.

## The interesting part: the cart optimizer

Given a wantlist, a set of listings per item across stores and Discogs sellers, and a
per-source shipping cost paid once per order, minimize:

```
total = Σ(listing prices) + Σ(shipping per source used, counted once)
```

This is a variant of the **uncapacitated facility location problem**, which is NP-hard.
Sources are facilities, shipping is the opening cost, and listing prices are the
assignment costs. Exact solving is not worth it at this size, so the optimizer runs a
greedy approximation:

1. Build a profile per source: shipping cost, plus the cheapest listing it offers for
   each item.
2. Repeatedly pick the source with the best `marginal cost / items covered` ratio, where
   marginal cost is its shipping plus its cheapest prices for everything still uncovered.
   Assign those items and mark them covered.
3. Run a cleanup pass. Once a source is in the cart its shipping is already paid, so for
   every assigned item, check whether another source already in the cart offers it
   cheaper. Reassign if so.

Step 3 is where most of the real savings come from. Greedy alone commits items to the
source that won them early, and the cleanup catches the ones that got cheaper the moment
a later source joined the cart for unrelated reasons.

Stores and Discogs marketplace sellers are treated as the same kind of object, so a cart
can mix a Berlin shop with three independent sellers without any special casing.

## Gem Score: taste intelligence

Scoring what is actually worth digging for, rather than what is popular. Each release gets
a 0 to 100 composite across five signals:

| Signal | What it measures |
|---|---|
| YouTube obscurity | Low view counts on an inverse log scale, so underground pressings rank up |
| YouTube engagement | Likes to views ratio, which finds small but passionate audiences |
| DJ validation | DJ names appearing in comments, a proxy for club play |
| Discogs rarity | Want to have ratio, sought after but scarce |
| Genre depth | Genre richness in the comment text |

Releases land in tiers: hidden gem, club weapon, deep cut, or known quantity. The tiering
matters more than the raw number, because a record you have never heard of scoring 80 is
a recommendation, and one scoring 80 that everyone owns is not.

## The rest of the system

**Wantlist scanning.** Discogs OAuth, then every wantlist item is checked against the
Discogs marketplace with live progress. The first version also checked 12 record stores:
EU stores scraped per item with Puppeteer, US stores mirrored locally (roughly 37,000 items
across Shopify and WooCommerce catalogs). Store scanning has been off by default since
September 2026, so the live product works from Discogs sellers, where the same
once-per-order shipping problem applies.

**Mix to Cart.** Paste a SoundCloud, YouTube, or Mixcloud link and it recovers the
tracklist from the description, then the comments, then audio fingerprinting if both fail,
and searches inventory for every track in the mix. Three fallbacks because DJ mixes label
their tracklists in three completely different ways depending on the platform.

**Discover.** Recommendations built from a taste profile weighted across Last.fm
similarity, SoundCloud likes and follows, YouTube history, and the Discogs wantlist.

**Daily rescans.** A background job re-checks the wantlist once a day and emails when
something new comes into stock. Most records that matter sell within days.

**WAXY.** A chat assistant over the catalog and the user's collection, built on the
Anthropic SDK with an MCP server so the same tools work from the app or an editor.

**Chrome extension.** Pulls listing data while browsing Discogs directly.

## Engineering notes

Roughly 1,000 commits and 66 test files. The tests are the reason this is maintainable:
scrapers break constantly when a store redesigns, and fixture-based parser tests catch it
before a user sees an empty result. Separate suites cover the optimizer, matcher,
shipping math, price freshness, and each store parser independently.

Record matching is its own hard problem. The same pressing is listed as "Artist - Title
(Label 001)" in one store and "ARTIST / TITLE 12\"" in another, so matching runs on
normalized artist and title with catalog number as a strong signal when present.

## Stack

Node, Express, SQLite (better-sqlite3), Puppeteer with stealth, Discogs API and
marketplace API, YouTube Data API, Last.fm, Anthropic SDK, MCP, Zod, PM2.
