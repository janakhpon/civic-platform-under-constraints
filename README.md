# Designing and Maintaining a Civic Platform Under Constraints

_A case study from both ends: the design, and the same system years later. The organisation running this platform could itself become a target, so we built it with no public API and no database reachable from the internet. Just files in storage, updated once a day. It held. The cost was that every byte of work moved onto the reader's phone, and nobody was watching that number. By then the main table was a 149 MB page, and many readers were on connections as slow as 400 kbps. This is the design, and what it cost._

![Article cover - designing and maintaining a civic platform](./assets/civic-platform-under-constraints.avif)

Most architecture arguments happen on the axis everyone enjoys: which framework, which database, which cloud. This one got settled on a different axis, and that is the part still worth writing down years later.

We were building a public data platform for a non-profit. Curated records, a searchable explorer, four dashboards, and one property that quietly decided everything else: **the organisation running it could itself become a target.**

Not the data. The data was meant to be public, freely reusable by anyone who credited the source. The organisation.

Once the operator is a target, infrastructure stops being only a cost. Every service you run is a surface, and every surface is something someone has to defend indefinitely.

I was a full-stack developer on the team that designed this. I proposed the architecture described here, and when the team agreed I led the implementation and built the CI/CD, the data pipeline, and the infrastructure. Years after it shipped, I did the migration and the optimisation.

This is long because it is two stories that only make sense together. The first is the design, and why the boring option won. The second is what that design cost years later, told in the order the work actually happened, including three instincts that were all wrong, an instrument that lied by two orders of magnitude, and a fix that made the scores worse before it made them better.

It runs across two pages, and the break is not between those two stories. It falls where the work changed shape.

**This page: the design, and the network.** Parts 1 to 12. The threat model, three architectures costed before any code, the bill that arrived years later, and getting 932 MB of cold-visit weight down to about 5.

**[Off the wire, onto the device](./docs/part2.md).** Parts 13 to 16, then where it stands. The bugs a compiler found once type checking went on, the render scores the optimisation made worse, the browser's one main thread, and the work I decided not to do.

The numbered parts run 1 to 16 straight through both pages, so a reference to Part 4 always means the section, never the page.

---

## Part 1: The box we were designing in

Four constraints. None of them negotiable.

**The threat model came first.** The data was public by design, so confidentiality was not the concern. Availability and integrity were.

A wrong number here is not a degraded user experience. It is a wrong number in somebody else's published reporting.

Correctness was not negotiable, and neither was staying reachable on the day it mattered most.

Worth being precise about what integrity meant in practice, because I was not precise about it at the time. Everything the design does for integrity defends against our own pipeline being wrong: row counts checked against the source's authoritative total, a publish that cannot be read half-applied, a bad build that degrades to the previous one rather than to a blank page.

None of that defends against someone who gains a write path into storage. A reader has no way to check that the bytes they received are the bytes we published; they trust the storage, and that is the whole of it. For a design whose whole premise is that the operator might be a target, that is a real omission, and the fix is a known one: sign what the pipeline publishes, so a reader verifies provenance rather than trusting the bucket. We defended the operator by removing services, and never extended the same reasoning to the storage we kept.

**The operating budget was small and fixed.** Not zero: somebody pays for object storage, for egress, and for the host that runs the internal database. But it was a figure the organisation had already committed to, with no line item that grows next quarter. Any design that added a *new* recurring bill was not a solution; it was a cost deferred onto someone with no way to absorb it.

**The readers were the constraint that reframed the problem.** Most are on slow mobile connections, roughly 400 kbps to 1.5 Mbps, on phones rather than laptops.

When your readers are on connections like that, bytes stop being one factor among several.

A framework can be fast or slow at the margins. A hundred megabytes is a hundred megabytes on any framework, and at 400 kbps it is a coffee break.

**The fourth constraint made the problem tractable, and it is the one teams miss.** The data updates once a day, from a human-curated source of record. Not once a second. Once a day.

Almost every expensive component in a conventional web stack exists to serve a read that must reflect a write from a moment ago. We did not have that requirement. Noticing it in time is what made the rest possible.

---

## Part 2: Three options, costed before any code

We wrote down three candidate architectures deliberately, before building any of them.

### Option one: the conventional stack

A CMS for data entry, a central database, a public API behind a gateway, a rendered frontend. Rate limiting, tokens, caching, a web application firewall, layers end to end.

This is the default reach, and on the merits it is a perfectly good design. It fails both hard constraints at once.

Serving thousands of readers through an API you operate means real compute, and real compute means a monthly bill that grows with success. That alone disqualified it.

The deeper problem was the surface. An API endpoint, a gateway, and a database all exist, all have addresses, and all need patching and monitoring indefinitely by whoever runs them. Each is a thing that can be taken down on the day the data matters most.

### Option two: the hybrid

Keep the CMS internal. Have a scheduled job read from a read-only replica, pre-render what can be pre-rendered, keep a small number of secure endpoints for the parts that resist it, and add a service worker plus a client-side store for offline support.

This was the close one, and it deserves a real reason rather than "we picked the simplest thing."

It genuinely removes most of the cost and most of the surface. But "a small number of secure endpoints" is still a server in the request path, and the request path was exactly what we wanted to empty. One endpoint still needs rate limiting, still needs its dependencies patched, still needs somebody reachable when it stops answering, and still puts a live target in front of a database.

Half a liability is not half the work. It is nearly all of the operational work for a fraction of the benefit, and it is the half you stop thinking about after six quiet months.

### Option three: nothing in the request path

No public API. No database reachable from the internet.

A curated database stays the internal source of record. A scheduled pipeline reads it, pre-computes exactly the files the frontend needs, and uploads them to object storage. The frontend is a static site that fetches those files directly from the browser.

Storage is the API.

```
  internal, unreachable from the web        public
 ┌──────────────────────────────┐        ┌──────────────────────────┐
 │  curated DB  ──►  pipeline   │  ────► │  object storage  ──► CDN │ ──► browser
 │  (source of      (scheduled, │ upload │  (chunks, shards,        │
 │   record)         once/day)  │        │   manifest, gzipped)     │
 └──────────────────────────────┘        └──────────────────────────┘
        no inbound path from here ▲             nothing executes here
```

Read that diagram for what is missing rather than what is present. There is no arrow from the public side back to the internal side, and there is no compute in the request path. Those two absences are the design.

We chose the third, and shipped it.

---

## Part 3: Why the boring option won

The honest reason is not elegance. It is that **option three deletes categories of problem instead of defending against them.**

There is no API to rate-limit, because there is no API. There is no public path from the internet to the database, because nothing in the request path can reach it. There is no origin server to overwhelm, because the origin is a storage bucket, which is the one component here that is genuinely good at absorbing a traffic spike.

The cost story falls out of the same decision rather than being a separate optimisation. The bill is storage, egress, and the host for the internal side, and it is a bill that stays roughly flat: there is no compute to pay for when nobody is reading, and nothing to autoscale when everybody is. Flat and predictable was the goal, not free.

The availability property is the one I underrated at design time. If the pipeline host dies, the site keeps serving the last good data, because the pipeline is not in the request path either.

The failure mode of the build system is "the data goes stale," not "the site goes down."

For a platform whose worst day is the day it is most needed, that distinction is the entire point.

Offline support then came almost free. Files that are immutable and cacheable are precisely what a service worker is good at holding. We were not so much adding offline capability as declining to prevent it.

---

## Part 4: What we accepted in exchange

Every one of those wins was bought. Being precise about the price is the part I would want to read in someone else's write-up.

**No fresh data, by construction.** A reader can never see anything newer than the last pipeline run. Because the requirement was one update per day, this cost us nothing. Had the requirement been minutes, option three would have been the wrong answer, and the design would have needed replacing rather than tuning.

**Query flexibility moved to the client.** With no query layer, anything a reader wants to filter or aggregate has to be computable from files we decided to ship in advance. Every new view becomes a pipeline change rather than a query. That is a slower iteration loop, and over time the pipeline accumulates knowledge about the frontend's needs, which is a coupling worth watching.

**All the weight lands on the client.** Pushing work out of the request path does not delete the work. It relocates it onto the reader's device, which in this audience is the least powerful machine in the whole system.

All three were understood when we chose, and none of them surprised anyone later. What we never did was attach a number to the third one.

That third one is the bill. It took years to arrive, and everything from here is what it cost.

---

## Part 5: Years later, the bill

The design held. It absorbed real growth with no architectural change: years of steadily accumulating records, and a readership in the thousands with the occasional sharp spike.

The operating cost stayed flat and predictable, which was the goal. And nothing ever paged anyone about a crashed process, an unpatched runtime, or a service that had stopped answering, because there was no service in the request path to crash. That is a narrower claim than saying nothing broke. Things broke, and later parts of this article are mostly about them. What the design removed was the class of failure that cannot wait until morning.

Then the tradeoff came to collect.

Records accumulated every day for years, and nothing in the design pushed back on file size. The main explorer table grew into the tens of thousands of rows.

I came back to a codebase several years old, extended by different hands through a period when shipping at all was the win. It worked in the narrow sense: the data was correct and the site stayed up.

It did not work in the sense that mattered. The people who most needed the data were the least able to load it.

Here is what a cold visit actually cost, measured rather than estimated:

| Page | Cold-visit weight | Requests |
|---|---|---|
| Data explorer | ~149 MB | 47 |
| Dashboard | ~570 MB | thousands |
| Listing page | ~209 MB | thousands |
| All four pages | 932 MB | — |

Readers waited ten to fifteen minutes for a working table. At 400 kbps, the honest lower bound for part of the audience, the arithmetic puts it near fifty minutes. I label that one a projection from measured bytes divided by stated bandwidth, because that is what it is.

Everything was still correct. Everything was still reachable. No alert had fired, because from the system's point of view nothing was wrong.

That is the failure mode of a decision that was right when it was made and never revisited. It does not announce itself with an incident. It shows up as a page that technically works and that a reader on a phone abandons.

The rest of this is what I did about it, under the same two walls: nothing added to the monthly bill, and no change to the architecture.

---

## Part 6: Three instincts, all wrong

Three fixes were on the table before any measurement. I held all three at some point. Measurement killed all three.

**Migrate the framework.** The frontend sat on Next.js 14, still on the Pages Router, with type checking switched off and a lot of dead code. Modernising it was overdue on its own merits, and the tempting story is that the site was slow because the framework was old.

I did not believe the story, so I measured before writing any migration code.

The explorer downloaded ~149 MB over 47 requests on every cold visit. Transferred size equalled raw object size, byte for byte, which is the signature of no compression at all.

The framework version appears nowhere in that story. Next.js 16 renders the same 149 MB no faster off the wire. Migrating alone would not have moved the number by a second.

**Sync incrementally from the source's changelog.** Have the pipeline poll the curated database's change feed and update only what moved. It sounds obviously right.

It fails on a detail: **deletes are invisible to polling.** A record removed at source simply stops appearing, and a diff-based sync has no event to react to. Webhooks would fix that and add attack surface to the one thing we had deliberately kept unreachable. And it optimises the pipeline, which was not the bottleneck.

**Put a query engine in the browser.** A WASM build of an embedded analytical database over a columnar file, the client issuing SQL and fetching byte ranges. Genuinely interesting for large read-only datasets.

Two things kill it here. The engine binary is several megabytes, on the order of the entire encoded dataset I was about to produce, so the first visit pays for the engine before reading a row. And WASM cold-start lands hardest on low-end Android phones, which is exactly the wrong place in this audience.

At tens of thousands of rows, an in-memory filter over a compact index does everything the page needs.

I wrote down the condition under which that verdict flips: roughly a tenfold growth in the dataset. **Naming the trigger that would make a rejected option correct is more useful than pretending it is wrong forever.**

What survived measurement was unglamorous: compress, then ship less, then put a CDN in front. All three independent of the migration.

**The check I would keep: for a page that feels slow, measure the wire before you touch the framework.** The bytes told me where to work in about an hour, and they were free to read.

---

## Part 7: The instrument lied

Before any of those numbers were worth quoting, the measurement itself had to survive scrutiny. It did not, the first time.

My first capture script decided the page had finished loading using a network-idle heuristic: wait until there have been no new requests for a short window, then call it done. A great deal of tooling works this way.

On this page that heuristic reported the ~149 MB explorer as 0.79 MB in 11 seconds.

The idle window elapsed during a lull mid-download. The script declared the page loaded and stopped counting while a hundred-plus megabytes were still streaming in.

A generic "page loaded" signal measures request timing, not bytes transferred, and on a heavy page those two diverge catastrophically.

I replaced the heuristic with evidence-based completion: a capture counts as done only when every data request has actually reached `loadingFinished` and the network has been quiet for at least ten seconds. Partial captures self-flag `complete: false` and list the unfinished URLs. Bytes are summed from the request ledger, including in-flight transfers.

Had I trusted the first tool, I would have written a confident report about a problem that was wrong by two orders of magnitude.

Two habits came out of that, and they are the reason I trust the rest of the numbers in this article.

**Keep a control page.** One page on the site fetches no bucket data. Measured across both deployments it came out at 1.63 MB against 1.66 MB, which tells me the environment did not drift between captures. Without a control, a before/after is a story about two runs rather than two builds.

That check has its own blind spot, and it is worth naming: a control page catches drift between runs, not a tool that is wrong the same way in both. The idle heuristic was exactly that kind of wrong, and no amount of controls would have caught it. Only changing the completion rule did.

**Quarantine bad captures, never delete them.** Three sets of measurements were invalidated during this work: the premature-idle captures, a flagged partial, and a batch of mixed-provenance render scores. Each sits in its own folder with a note explaining the failure mode. Deleting them would have made the archive look cleaner and the conclusions less trustworthy.

---

## Part 8: The cheapest ninety-three percent

Compression came first, because it was the highest-impact change that touched nothing structural.

Two misconceptions were in the way, and both are common.

**"The server or the CDN compresses for me."** S3 does not. It is dumb storage: it returns exactly the bytes you stored, echoes only the headers you set, does no on-the-fly compression, and ignores the client's `Accept-Encoding` entirely.

**"Smaller is always better."** It is not, when the origin cannot negotiate.

On top of no compression, the files were pretty-printed, carrying indentation a human reads and a machine does not need. For this dataset pretty-printing alone inflated the wire by about 56 percent: 112.5 MB served against 72.1 MB written compactly, before real compression entered the picture.

Then gzip at maximum level removed 93.0 percent of the compact payload, taking that 72.1 MB file to ~5 MB. Measured on the real files.

JSON with this much repeated structure, the same field names on every one of tens of thousands of records, compresses extremely well. Gzip is very good at exactly that redundancy.

### The better option I turned down

Brotli at q11 measured 96.2 percent reduction against gzip's 93.0, roughly 46 percent smaller again on the same file.

I did not use it, and the reason is the origin, not the merits.

Because S3 stores one representation and serves it to everyone regardless of what the client says it accepts, a stored Brotli object gets served to clients that cannot decode it. A modern browser is fine. An older Android WebView, a minimal embedded HTTP client, or a plain command-line fetch may not be, and a mixed, low-end audience includes exactly those.

**When the origin cannot negotiate, you ship the encoding every client can read.** Gzip has had that for twenty-plus years.

The nuance worth keeping: Brotli is still available, just not from the origin. A CDN *does* negotiate per client, so Brotli stays available as an on-the-fly layer at the edge without ever storing a non-universal encoding at the origin. The rule is about where negotiation happens, not about which algorithm is better.

So the pipeline learned to write compact JSON and upload it gzipped with `ContentEncoding: 'gzip'` set on the put. Browsers decode transparently, so the frontend did not change at all: `response.json()` just works.

No new service, no new cost, architecture untouched. The pipeline produces different bytes; everything downstream is identical.

One objection deserves a number rather than a reassurance, because "you moved the cost to the client's CPU" is the obvious comeback. Decompressing the entire index in the browser measured 21 milliseconds, with `JSON.parse` at 108 ms. Negligible against the minutes it saves.

---

## Part 9: Shipping less, not just smaller

Compression got the payload to about 5 MB, and 5 MB is still a lot on a 400 kbps phone. Worse, most of those bytes were paying for content the visitor had not asked to see.

Each row carried more than the fields the table displays. It also held long free-text fields. The table shows twenty-odd short columns. That free text is what a reader sees only after opening one record.

Every visitor downloaded every long text field for every record, up front, to render a paginated table showing twenty rows at a time.

### Three shapes instead of one file

**A compact index.** Only the short fields the table and its filters use, flattened, with categorical values dictionary-encoded. There are only about 150 to 250 distinct values across every categorical field in the whole dataset, so each is stored once in the manifest and referenced by a short key rather than repeating the full string on every row.

That index is ~2.6 MB gzipped, covering every row in the table.

**256 detail shards.** The long text fields, sliced by a hash of the record id, fetched only when a reader opens that record.

**A manifest**, 4.6 KB, naming the current set of chunks and shards.

Freshness lives in a 4.6 KB pointer while the bulk stays immutable. That sentence is the whole design.

### Why the chunk names matter more than the chunk sizes

The index is not one ~2.6 MB file. It is 23 chunks, each named after a hash of its own contents, median size around 101 KB.

Because the name is a content hash, a chunk's URL only changes when its bytes change. That lets me serve chunks as immutable with a one-year max-age. A returning reader does not re-download them at all.

This is also where the naive version of the same idea quietly defeats itself, and the distinction is the most transferable thing in this section.

Cut the index into chunks of a *fixed row count*, and inserting one new record near the front shifts every row after it into a different chunk. Every one of those chunks gets new bytes, every hash changes, and a returning reader re-downloads the entire index because one record was added.

That is the cache-busting failure mode fixed-size chunking walks straight into.

Instead the boundaries are **content-defined.** I walk the sorted rows and start a new chunk when a hash of the current row's key meets a boundary condition, clamped to a minimum and maximum so no chunk is pathologically small or large.

Because a boundary is a property of the row's own content, appending the day's new records disturbs only the one or two chunks those records fall into. The other twenty-one hash to the same names they had yesterday and stay in every reader's cache.

That is the difference between a design that is incremental on paper and one that is incremental in practice.

### Publishing without a torn read

The old pipeline overwrote thirty-odd files in place over about fifteen seconds. A visitor who loaded the page during that window could get a mix of old and new files, a torn read, with no signal anything was wrong.

The chunked design fixes this structurally rather than with locking. All new chunks and shards upload first, under names nothing references yet. The manifest uploads last, in a single write, only after every chunk it points to is in place.

Until that final write, the site serves the previous manifest and the previous chunks, all still present. The switch is one object.

If the run dies halfway it dies before the manifest flips, and the live site never sees the half-finished state.

The same property gives incremental publishing for free. On a run where nothing changed, every chunk hashes to a name already in the bucket, so a cheap existence check skips all of them: a re-run with no data change uploaded nothing at all and skipped every object.

A thirty-day garbage collection sweep removes chunks nothing references any more.

### The fallback I drilled before trusting any of it

If the manifest fails to load, or a chunk 404s, or the contract version mismatches, the client falls back silently to the old full-file path.

I drilled it by deleting the manifest and watching the page take a 403, issue four legacy fetches, and render correctly. It was then field-tested for real by an unrelated deploy hiccup, which it absorbed without anyone noticing.

A performance optimisation that can take the site down is not one I want in front of this audience.

### What repeat visits cost now

Zero network requests. Twenty-five of twenty-five resources from disk cache, page complete in 0.6 seconds. After the TTL expires, revalidation returns 304s and costs zero bytes.

That is the payoff of immutability, and it is the reason I did not build the more sophisticated thing.

### The delta-sync design I rejected

Row-level change files and versioned snapshots, so a returning client fetches only rows that changed. A real technique that would ship the fewest possible repeat-visit bytes.

It also carries version chains, a compaction strategy so delta history does not grow forever, and stale-client edge cases where a reader who has been away long enough must be detected and rebased onto a fresh snapshot.

Immutable content-hashed chunks already make repeat visits free, as the 0.6 seconds above shows. For a small pipeline, the simpler design you can reason about at 2am beats the optimal one that needs a specialist to debug.

---

## Part 10: The seam that had to hold

Everything above spans two repositories that share no code: a Node pipeline that publishes, and a Next.js frontend that consumes. Different package managers, different deploy cadences, different people plausibly touching them on different days.

A design like this fails at that seam. The pipeline changes a field, the frontend keeps parsing the old shape, and nothing errors: it just renders slightly wrong numbers, which on this platform is the failure that matters most.

So the seam is an explicit, versioned contract rather than a shared assumption.

The manifest carries a `contractVersion` and the expected row count. The frontend refuses an index whose version it was not built for, or whose row count looks wrong, and falls back rather than rendering against a shape it does not understand. The hash function that assigns records to shards is byte-identical on both sides, and both agree on 256 shards, because a one-line divergence there would silently send readers to the wrong detail file.

I verified that at line level across both repositories rather than assuming it: same contract version, same hash implementation, same shard count, same manifest shape, plus the row-count assertion. Then I proved it end to end on real hosting.

The general shape is worth stealing: **when two systems must agree and cannot share code, version the agreement and let the consumer refuse.** A contract that can only be honoured by convention will eventually be broken by someone who never read the convention.

## Part 11: The 570 MB that was not a big dataset

The dashboard was the heaviest page at 570 MB, and I assumed for a while that it was simply carrying more data than the explorer.

Decomposing it reframed the problem completely.

The dashboard downloaded the same records five times over, once per grouping dimension: a date field and four categorical ones. Each copy carried full records rather than references.

The dashboard's 570 MB was not a large dataset. It was one dataset duplicated by the grouping strategy.

That distinction changes what counts as a fix. Compression hides the redundancy: gzip took ~570 MB of JSON to 27 MB, which looks like a triumph and leaves the modelling error entirely in place. Reaching for a bandwidth band-aid when the real issue is data modelling is a mistake I have watched teams make repeatedly, and the compression number is exactly what makes it easy to miss.

The decomposition also corrected a second assumption. I had been calling the dashboard's problem an "image storm," because it probed thousands of images on load. Splitting the transfer by type settled it: all but about 4 MB of that ~570 MB was JSON. The images were a rounding error on the byte total.

The images were a *request-count* problem, thousands of round trips, not a byte problem. Two different costs needing two different fixes, and I would have aimed at the wrong one.

The structural fix, having the pipeline emit lean groupings where each group is a list of ids plus one shared store of records, is scoped and not done. It needs a pipeline order-field to preserve the exact visuals, which gated it. Doing that de-duplication in the browser instead would add main-thread work, which [Part 14](./docs/part2.md#part-14-what-the-optimization-broke) explains is the last thing this system needs.

---

## Part 12: A content network that added no new bill

The payload was now small, but every byte still came from a single storage region, and a request from where the readers are to a bucket in another region pays that round trip on every uncached fetch.

A CDN is the standard answer and the standard worry is that it adds a bill. Here it did not.

I put a CloudFront distribution in front of the existing bucket. Nothing about the data moved. The origin stayed the same bucket, the pipeline kept uploading to the same place, and an origin access control let me then close the bucket to direct public reads and force all traffic through the distribution.

At this site's traffic it sits inside the provider's standing free allowance, a terabyte of egress and ten million requests a month, with origin fetches into the distribution unbilled.

So the delivery layer added **no new monthly cost**, and it stays that way until roughly 670,000 explorer visits a month, which is far beyond current traffic.

I want to be careful with that claim rather than round it down to "free," because two things are true. The platform does cost money: storage, egress, and the machine running the internal database are all real line items, and none of this work removed them. What the CDN did was avoid *adding* to them.

And an allowance is not a promise. It has a ceiling, that ceiling is the number above, and a provider can change its terms. Writing the break-even down is the difference between a cost decision someone can re-check later and a claim that quietly expires.

Verified end to end: 45 of 45 edge cache hits over HTTP/3, and the same payload either way, 5.16 MB through the CDN against 5.18 direct from storage.

### The alternative I looked at and declined

An object store with no egress fees, behind its own edge network.

I did not choose it, and the reason is the constraint rather than the merits. Adopting it meant migrating or dual-publishing every public object and rewriting hardcoded bucket URLs scattered through two codebases. That is architectural churn on a system I was told not to churn, for a latency gain that is small next to the payload win already banked.

Worth naming the pattern underneath, because it comes up constantly: the popular claim that one storage product is "faster" than another is usually, underneath, a claim about having a CDN versus not having one. Once a CDN sits in front of the existing bucket, most of that gap closes without moving any data.

If traffic ever grows into real egress bills the calculus changes and I would revisit. Today it would be motion without payoff.

### Two console traps

The first is the price class. The cheaper classes serve from only a subset of edge locations, so check that the one you pick covers where your readers actually are before taking the saving.

The other cost me an afternoon: **a HEAD request does not populate the edge cache.** I verified cache behaviour with a command that issues HEADs, saw an unbroken run of misses, and briefly believed the CDN was misconfigured.

It was fine. The verification was wrong. Real GET traffic populated and served from the edge exactly as it should.

Measure the thing the way the browser actually does it, or the measurement lies to you. Second time in this project that same lesson arrived.

---

## Continue

That is the network side of it. The payload is small, it is close to the reader, and none of it added a line to the monthly bill.

The second half is what happened once the bytes stopped being the problem: the migration I had deferred, the bugs a compiler found the moment type checking went on, the render scores the optimisation made *worse* before it made them better, and the browser's one main thread that everything now competed for.

**[Off the wire, onto the device →](./docs/part2.md)**
