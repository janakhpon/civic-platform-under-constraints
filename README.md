# Designing and Maintaining a Civic Platform Under Constraints

_A case study from both ends: the design, and the same system four years later. The organisation running this platform could itself become a target, so we built it with no public API and no database reachable from the internet. Just files in storage, updated once a day. It held. The cost was that every byte of work moved onto the reader's phone, and nobody was watching that number. By then the main table was a 149 MB page, and most readers were on 400 kbps. This is the design, and what it cost._

![Article cover - designing and maintaining a civic platform](./assets/civic-platform-under-constraints.avif)

Most architecture arguments happen on the axis everyone enjoys: which framework, which database, which cloud. This one got settled on a different axis, and that is the part still worth writing down years later.

We were building a public data platform for a small non-profit. Curated records about individuals, a searchable explorer, four dashboards, and one property that quietly decided everything else: **the organisation running it could itself become a target.**

Not the data. The data was meant to be public, freely reusable by anyone who credited the source. The organisation.

Once the operator is a target, infrastructure stops being only a cost. Every service you run is a surface, and every surface is something a very small technical team has to defend indefinitely.

I was a full-stack developer on the team that designed this. I proposed the architecture described here, and when the team agreed I led the implementation and built the CI/CD, the data pipeline, and the infrastructure. Years after it shipped, I did the migration and the optimisation, and that half I did alone.

This is long because it is two stories that only make sense together. The first is the design, and why the boring option won. The second is what that design cost years later, told in the order the work actually happened, including three instincts that were all wrong, an instrument that lied by two orders of magnitude, and a fix that made the scores worse before it made them better.

---

## Part 1: The box we were designing in

Four constraints. None of them negotiable.

**The threat model came first.** The data was public by design, so confidentiality was not the concern. Availability and integrity were.

A wrong number here is not a degraded user experience. It is a wrong number in somebody else's published reporting.

Correctness was not negotiable, and neither was staying reachable on the day it mattered most.

Worth being precise about what integrity meant in practice, because I was not precise about it at the time. Everything the design does for integrity defends against our own pipeline being wrong: row counts checked against the source's authoritative total, a publish that cannot be read half-applied, a bad build that degrades to the previous one rather than to a blank page.

None of that defends against someone who gains a write path into storage. A reader has no way to check that the bytes they received are the bytes we published; they trust the storage, and that is the whole of it. For a design whose whole premise is that the operator might be a target, that is a real omission, and the fix is a known one: sign what the pipeline publishes, so a reader verifies provenance rather than trusting the bucket. We defended the operator by removing services, and never extended the same reasoning to the storage we kept.

**The operating budget was small and fixed.** Not zero: somebody pays for object storage, for egress, and for the host that runs the internal database. But it was a figure a small non-profit had already committed to, with no line item that grows next quarter. Any design that added a *new* recurring bill was not a solution; it was a cost deferred onto someone with no way to absorb it.

**The readers were the constraint that reframed the problem.** Most are on throttled mobile connections, roughly 400 kbps to 1.5 Mbps, on phones rather than laptops, in a region with frequent power and connectivity interruption. Some arrive over VPNs, which narrows the pipe further.

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

The deeper problem was the surface. An API endpoint, a gateway, and a database all exist, all have addresses, and all need patching and monitoring indefinitely by a team with no security engineer on it. Each is a thing that can be taken down on the day the data matters most.

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
 │   record)         twice/day) │        │   manifest, gzipped)     │
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

The design held. It absorbed real growth with no architectural change: past 100,000 records across the main tables, around 132,000 events and roughly 10,000 active readers a year, peaking near 2,800 daily during a spike.

The operating cost stayed flat and predictable, which was the goal. And nothing ever paged anyone about a crashed process, an unpatched runtime, or a service that had stopped answering, because there was no service in the request path to crash. That is a narrower claim than saying nothing broke. Things broke, and later parts of this article are mostly about them. What the design removed was the class of failure that cannot wait until morning.

Then the tradeoff came to collect.

Records accumulated at twenty to twenty-five a day for five and a half years, and nothing in the design pushed back on file size. The main explorer table passed about 40,000 rows.

I came back to a codebase about three years old, extended by several hands through a period when shipping at all was the win. It worked in the narrow sense: the data was correct and the site stayed up.

It did not work in the sense that mattered. The people who most needed the data were the least able to load it.

Here is what a cold visit actually cost, measured rather than estimated:

| Page | Cold-visit weight | Requests |
|---|---|---|
| Data explorer | ~149 MB | 47 |
| Dashboard | ~570 MB | 3,413 |
| Profile listing | ~209 MB | 3,875 |
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

At forty thousand rows, an in-memory filter over a compact index does everything the page needs.

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

JSON with this much repeated structure, the same field names on every one of forty thousand records, compresses extremely well. Gzip is very good at exactly that redundancy.

### The better option I turned down

Brotli at q11 measured 96.2 percent reduction against gzip's 93.0, roughly 46 percent smaller again on the same file.

I did not use it, and the reason is the origin, not the merits.

Because S3 stores one representation and serves it to everyone regardless of what the client says it accepts, a stored Brotli object gets served to clients that cannot decode it. A modern browser is fine. An older Android WebView, a stripped-down VPN client, or a plain command-line fetch may not be, and this audience has exactly those in the mix.

**When the origin cannot negotiate, you ship the encoding every client can read.** Gzip has had that for twenty-plus years.

The nuance worth keeping: Brotli is still available, just not from the origin. A CDN *does* negotiate per client, so Brotli stays available as an on-the-fly layer at the edge without ever storing a non-universal encoding at the origin. The rule is about where negotiation happens, not about which algorithm is better.

So the pipeline learned to write compact JSON and upload it gzipped with `ContentEncoding: 'gzip'` set on the put. Browsers decode transparently, so the frontend did not change at all: `response.json()` just works.

No new service, no new cost, architecture untouched. The pipeline produces different bytes; everything downstream is identical.

One objection deserves a number rather than a reassurance, because "you moved the cost to the client's CPU" is the obvious comeback. Decompressing the entire index in the browser measured 21 milliseconds, with `JSON.parse` at 108 ms. Negligible against the minutes it saves.

---

## Part 9: Shipping less, not just smaller

Compression got the payload to about 5 MB, and 5 MB is still a lot on a 400 kbps phone. Worse, most of those bytes were paying for content the visitor had not asked to see.

Each row carried more than the fields the table displays. It also held long bilingual narrative fields: the detailed account of each case in two languages. The table shows twenty-odd short columns. The narrative is what a reader sees only after opening one record.

Every visitor downloaded every narrative for every record, up front, to render a paginated table showing twenty rows at a time.

### Three shapes instead of one file

**A compact index.** Only the short fields the table and its filters use, flattened, with categorical values dictionary-encoded. There are only about 150 to 250 distinct values across every region, sector, cause, and age bracket in the whole dataset, so each is stored once in the manifest and referenced by a short key rather than repeating the full string forty thousand times.

That index is ~2.6 MB gzipped for all about 40,000 rows.

**256 detail shards.** The long narratives, sliced by a hash of the record id, fetched only when a reader opens that record.

**A manifest**, 4.6 KB, naming the current set of chunks and shards.

Freshness lives in a 4.6 KB pointer while the bulk stays immutable. That sentence is the whole design.

### Why the chunk names matter more than the chunk sizes

The index is not one ~2.6 MB file. It is 23 chunks, each named after a hash of its own contents, median size around 101 KB.

Because the name is a content hash, a chunk's URL only changes when its bytes change. That lets me serve chunks as immutable with a one-year max-age. A returning reader does not re-download them at all.

This is also where the naive version of the same idea quietly defeats itself, and the distinction is the most transferable thing in this section.

Cut the index into chunks of a *fixed row count*, and inserting one new record near the front shifts every row after it into a different chunk. Every one of those chunks gets new bytes, every hash changes, and a returning reader re-downloads the entire index because one record was added.

That is the cache-busting failure mode fixed-size chunking walks straight into.

Instead the boundaries are **content-defined.** I walk the sorted rows and start a new chunk when a hash of the current row's key meets a boundary condition, clamped to a minimum and maximum so no chunk is pathologically small or large.

Because a boundary is a property of the row's own content, appending the day's twenty-odd new records disturbs only the one or two chunks those records fall into. The other twenty-one hash to the same names they had yesterday and stay in every reader's cache.

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

Immutable content-hashed chunks already make repeat visits free, as the 0.6 seconds above shows. For a solo-maintained pipeline, the simpler design one person can reason about at 2am beats the optimal one that needs a specialist to debug.

---

## Part 10: The two seams that had to hold

### The contract between two codebases

Everything above spans two repositories that share no code: a Node pipeline that publishes, and a Next.js frontend that consumes. Different package managers, different deploy cadences, different people plausibly touching them on different days.

A design like this fails at that seam. The pipeline changes a field, the frontend keeps parsing the old shape, and nothing errors: it just renders slightly wrong numbers, which on this platform is the failure that matters most.

So the seam is an explicit, versioned contract rather than a shared assumption.

The manifest carries a `contractVersion` and the expected row count. The frontend refuses an index whose version it was not built for, or whose row count looks wrong, and falls back rather than rendering against a shape it does not understand. The hash function that assigns records to shards is byte-identical on both sides, and both agree on 256 shards, because a one-line divergence there would silently send readers to the wrong detail file.

I verified that at line level across both repositories rather than assuming it: same contract version, same hash implementation, same shard count, same manifest shape, plus the row-count assertion. Then I proved it end to end on real hosting.

The general shape is worth stealing: **when two systems must agree and cannot share code, version the agreement and let the consumer refuse.** A contract that can only be honoured by convention will eventually be broken by someone who never read the convention.

### Designing for the person who inherits it

I inherited this codebase once, which is the cheapest possible education in what to leave behind.

Read together, several decisions above are the same decision. The automatic fallback means a bad publish degrades to the old experience instead of a blank page. The fail-loud pipeline means a partial fetch stops rather than shipping a smaller number. The contract version means a shape change is refused rather than mis-rendered. The content-hashed names mean a torn read is structurally impossible instead of merely unlikely. The rejected delta-sync design lost specifically because it needed a specialist to debug at 2am, and this system does not have one on call.

None of those make the site faster. All of them make it harder for the next person, including future me, to break it quietly.

That is also the honest answer to why the simpler option keeps winning here. It is not an aesthetic preference for simplicity. A system maintained by one or two people has a hard ceiling on how much cleverness it can carry, and designing above that ceiling is a way of handing someone a bill they did not agree to.

## Part 11: The 570 MB that was not a big dataset

The dashboard was the heaviest page at 570 MB, and I assumed for a while that it was simply carrying more data than the explorer.

Decomposing it reframed the problem completely.

The dashboard downloaded the same 31,000 records five times over, once per grouping dimension: year-month, sector, age, state, gender. Each copy carried full records rather than references.

The dashboard's 570 MB was not a large dataset. It was one dataset duplicated by the grouping strategy.

That distinction changes what counts as a fix. Compression hides the redundancy: gzip took ~570 MB of JSON to 27 MB, which looks like a triumph and leaves the modelling error entirely in place. Reaching for a bandwidth band-aid when the real issue is data modelling is a mistake I have watched teams make repeatedly, and the compression number is exactly what makes it easy to miss.

The decomposition also corrected a second assumption. I had been calling the dashboard's problem an "avatar storm," because it probed around 3,300 images on load. Splitting the transfer by type settled it: all but about 4 MB of that ~570 MB was JSON. The images were a rounding error on the byte total.

The avatars were a *request-count* problem, roughly 3,300 round trips, not a byte problem. Two different costs needing two different fixes, and I would have aimed at the wrong one.

The structural fix, having the pipeline emit lean groupings where each group is a list of ids plus one shared store of records, is scoped and not done. It needs a pipeline order-field to preserve the exact visuals, which gated it. Doing that de-duplication in the browser instead would add main-thread work, which Part 14 explains is the last thing this system needs.

---

## Part 12: A content network that added no new bill

The payload was now small, but every byte still came from a single storage region, and a request from the readers' region to a bucket in a neighbouring one pays that round trip on every uncached fetch.

A CDN is the standard answer and the standard worry is that it adds a bill. Here it did not.

I put a CloudFront distribution in front of the existing bucket. Nothing about the data moved. The origin stayed the same bucket, the pipeline kept uploading to the same place, and an origin access control let me then close the bucket to direct public reads and force all traffic through the distribution.

At this site's traffic it sits inside the provider's standing free allowance, a terabyte of egress and ten million requests a month, with origin fetches into the distribution unbilled.

So the delivery layer added **no new monthly cost**, and it stays that way until roughly 670,000 explorer visits a month, which is far beyond current traffic.

I want to be careful with that claim rather than round it down to "free," because two things are true. The platform does cost money: storage, egress, and the machine running the internal database are all real line items, and none of this work removed them. What the CDN did was avoid *adding* to them.

And an allowance is not a promise. It has a ceiling, that ceiling is the number above, and a provider can change its terms. Writing the break-even down is the difference between a cost decision someone can re-check later and a claim that quietly expires.

Verified end to end: 45 of 45 edge cache hits over HTTP/3, and the same payload either way, 5.16 MB through the CDN against 5.18 direct from storage.

### The alternative I looked at and declined

An object store with no egress fees behind its own network, which also has an edge location physically closer to this audience.

I did not choose it, and the reason is the constraint rather than the merits. Adopting it meant migrating or dual-publishing every public object and rewriting hardcoded bucket URLs scattered through two codebases. That is architectural churn on a system I was told not to churn, for a latency gain that is small next to the payload win already banked.

Worth naming the pattern underneath, because it comes up constantly: the popular claim that one storage product is "faster" than another is usually, underneath, a claim about having a CDN versus not having one. Once a CDN sits in front of the existing bucket, most of that gap closes without moving any data.

If traffic ever grows into real egress bills the calculus changes and I would revisit. Today it would be motion without payoff.

### Three console traps

One default is worth knowing, because the cheapest price class excludes the region this audience is in, so the cheap option is the wrong option here.

The other cost me an afternoon: **a HEAD request does not populate the edge cache.** I verified cache behaviour with a command that issues HEADs, saw an unbroken run of misses, and briefly believed the CDN was misconfigured.

It was fine. The verification was wrong. Real GET traffic populated and served from the edge exactly as it should.

Measure the thing the way the browser actually does it, or the measurement lies to you. Second time in this project that same lesson arrived.

---

## Part 13: The migration, and the bugs a compiler found

With the delivery path fixed, the migration I had deferred was worth doing, now honestly framed as maintainability work rather than a performance fix.

I chose a clean rebuild on the App Router rather than an in-place 14 to 15 to 16 upgrade. The old code health made the rebuild cheaper than the upgrade path, which is not the usual answer and was the right one here.

Next.js 16, React 19, strict TypeScript, every route ported behaviour for behaviour. Two details would catch anyone doing the same move on Next.js 16: the internationalisation library we were on does not support the App Router, which meant rewriting every message call site, and the middleware entry point is renamed.

Routing behaviour stayed identical, the default language unprefixed and the second under its own path segment, because changing user-visible URLs on a site other people link to is its own kind of breakage.

The change that paid for the whole migration was **turning type checking on.** The old build had `ignoreBuildErrors` set, which is how a three-year-old codebase ends up shipping bugs a compiler would have caught on the first build.

The clearest example: the explorer's Reset button threw a `ReferenceError` on every single click. It called a setter that never existed. With type checking off the build never complained, and because the error only fired on click, it survived in production for years.

### Three pipeline defects worth naming plainly

These are the kind that hide in a system that reports success too easily.

**The pipeline could publish a truncated dataset and report success.** If a page of the source fetch failed after its retries, the code caught the error, logged it, and carried on, and the top-level handler exited zero. A partial dataset would publish over the good one, with undercounted numbers, and the scheduled job would go green.

On a platform whose figures other people rely on, silently shipping a smaller number is the worst class of bug available to it.

The fix was to fail loudly: throw when the fetched row count does not match the source's authoritative total, and set a non-zero exit code in the top-level handler so the CI chain halts and the publish step never runs on a bad fetch.

A pipeline that stops is safe. A pipeline that continues past an error and publishes anyway is not.

**The second defect had already reached production.** One side of a verified-versus-unverified split was publishing as zero, so an entire category was presented under the wrong label.

The cause was one level of unwrapping. The source stores certain fields as a small object with the value nested inside, and the pipeline read one level too shallow, comparing an object against a string that never matched.

The reason it went unnoticed is the instructive part. The frontend read the same field correctly, because its own parser happened to unwrap twice. The interactive table showed the right thing while the summary count showed zero.

Two code paths reading the same source with different assumptions is a bug waiting for one of them to be right by accident.

**The third was a security defect.** The pipeline authenticates to the source with a token and followed the source's pagination links verbatim. Those links came back over plain HTTP even though the base URL was HTTPS, because of a misconfiguration in how the database advertised its own address behind its reverse proxy.

So on every paginated fetch the token went out in cleartext on the first leg. The redirect to HTTPS did happen, but only after the unencrypted request had already left with the authorization header attached.

Two-part fix, one immediate and one operational: force the pagination URLs to HTTPS in the fetch code, and rotate the exposed token.

While I was in there I also turned the index parity check from a warning into a hard failure, and stopped the build from stripping `console.error` and `console.warn` in production, which had quietly removed the observability anyone would need to diagnose the above.

The honest summary: the migration did not move the bytes number by a single kilobyte. It was the vehicle that made the codebase maintainable and surfaced real bugs, and it was worth doing for those reasons.

It was not the performance fix, and I would not let it be sold as one.

---

## Part 14: What the optimization broke

This is the part I would most want another engineer to read, because it is the part easiest to leave out of a case study.

Here is the before and after, both live production-built deployments, captured the same day on the same machine:

| Page | Before | After | Change |
|---|---|---|---|
| Explorer | ~149 MB | ~5.2 MB | −96.5% |
| Dashboard | ~570 MB | ~29 MB | −94.9% |
| Profile listing | ~209 MB | ~13 MB | −93.9% |
| All four pages | 932 MB | 49 MB | −94.8% |

Explorer time-to-loaded on the same machine and network went from 42 seconds to about 6, with first contentful paint at 244 ms. Projected against reader bandwidth, first-visit table data goes from ~13.3 minutes to ~14 seconds at 1.5 Mbps, and from ~50 minutes to ~53 seconds at 400 kbps.

Those are instrument numbers, and instrument numbers are not the point of this system. The one that matters came from someone opening the dashboard on their own machine and their own connection, without a capture tool involved: **about four to five minutes minimum, down to about forty-five seconds.**

I checked it against the instrumented values rather than just accepting the good news, because a pleasing anecdote is the easiest thing in the world to publish. It reconciles: the old dashboard measured 4 m 40 s at 570 MB, and the new one at ~28 MB is roughly 14 seconds of transfer at that bandwidth. The remaining thirty seconds is almost entirely an artifact of the dev environment, where about 3,300 avatar requests fail against a bucket that has no profile images. On production, where those images exist and cache, and with the lazy-loading fix taking the request count from ~3,300 to ~30, that tail should mostly disappear.

So the honest form of the claim is that a real reader went from abandoning the page to using it, on the worse of the two environments.

And the honest limit is that this is one reader on one connection, reporting their own experience. It is a signal, not a distribution, and I would not quote it as a median. What makes it worth including is that it agrees with the instrumented numbers rather than replacing them. Real-user data across the audience is the measurement I still do not have.

And then the scores got worse.

Lighthouse performance on the explorer went 77 to 42 on desktop. Dashboard total blocking time went 70 ms to 2,831 ms on desktop, and 88 to 4,135 on mobile.

Those are measured numbers, and they moved the wrong way.

### Why the old scores were flattering, and the new ones honest

Two things were happening at once, and separating them is the difference between an excuse and an explanation.

The first is a real cost. Cutting the payload by 95 percent made the site arrive fast and then revealed work the network had been hiding. **The bottleneck moved from the wire to the browser's main thread**: parsing the index, assembling the client-side filter structure, hydrating a React 19 tree, drawing the visualisations.

The second is a measurement artifact, and it explains why the old number looked good. Lighthouse's measurement window largely elapsed while 149 MB was still downloading. The old site's heavy client-side JavaScript had not run yet, so it was never counted. The new site loads fast enough that the same JavaScript now runs *inside* the window and gets measured.

The old TBT was not low. It was unobserved.

That distinction matters because reporting only the improvement would have been cherry-picking, and reporting only the regression would have been wrong about the cause.

### The browser has one main thread

While it is busy running JavaScript it cannot paint, scroll, or respond to a tap. The browser flags any task over 50 ms as a long task, and total blocking time is how much of that accumulates.

The explorer had one bad offender: a single synchronous function building the crossfilter index over all about 40,000 rows in roughly 747 milliseconds of uninterrupted work.

For three quarters of a second the page was frozen solid.

---

## Part 15: Two ways to protect one thread

### The fix I nearly wrote, and why profiling killed it

My instinct was virtualization: render only the visible table rows. It is the standard answer for a large table and I was ready to write it.

I profiled first, and the profile refused it. **The table was already paginated to about twenty DOM rows.** Virtualization would have been a risky rewrite solving a problem the table did not have.

The cost sat in that one synchronous build.

A Web Worker was the other candidate, and it was the heavy option: moving crossfilter off-thread entirely means serialising data across the boundary and restructuring how the explorer talks to its index. Real work, real risk.

So I time-sliced instead. The row-processing loop is broken into slices of about 24 milliseconds, and after each slice the function yields the thread back before resuming. No slice crosses the 50 ms line, so the browser can paint and handle input in the gaps.

This does not make the work faster. It is the same total CPU. It makes the work non-blocking, which is a different property and the one that matters for responsiveness.

The single step that genuinely cannot be sliced, crossfilter's own internal build, is isolated into its own task so it does not merge with the rest.

### The mechanism, because it is easy to fake

It is easy to write something that looks like yielding and is not.

**Awaiting a resolved promise does not yield.** The microtask runs before the browser gets a turn to paint, so the thread never actually goes back. A `yieldToMain` helper whose fallback is `Promise.resolve().then(...)` silently does nothing at all, and the "chunked" loop keeps blocking exactly as before.

To hand the thread over you have to end the current task and let the event loop cycle. `setTimeout(fn, 0)` does that, but nested timers get clamped to about 4 ms, which is pure overhead paid on every slice. A `MessageChannel` post resolves on the next task with effectively no clamp, so that is the primary path with the timer kept only as a non-browser fallback:

```ts
const SLICE_MS = 24; // well under the 50ms long-task threshold

function yieldToMain(): Promise<void> {
  return new Promise((resolve) => {
    if (typeof MessageChannel !== 'undefined') {
      const channel = new MessageChannel();
      channel.port1.onmessage = () => resolve();
      channel.port2.postMessage(undefined);
    } else {
      setTimeout(resolve, 0); // non-browser fallback
    }
  });
}
```

The loop itself is then unremarkable, which is the point. It yields on a time budget rather than a row count, so it adapts to whatever device it is running on instead of assuming a speed:

```ts
let sliceStart = nowMs();
for (let i = 0; i < dataSet.length; i++) {
  parsedData.push(parseSingleRow(dataSet[i], i, filters, ctx));

  if (nowMs() - sliceStart > SLICE_MS) {
    await yieldToMain();
    sliceStart = nowMs();
  }
}

// Start the crossfilter build in a fresh task rather than on the
// tail of the last parse slice, so the two cannot merge into one
// long task.
await yieldToMain();
```

Budgeting by elapsed time rather than by row count matters more than it looks. A fixed "yield every 500 rows" is tuned to one machine: on a fast laptop it yields far more often than needed, and on the low-end phone that actually needs protecting it produces slices well over the 50 ms line. Measuring the clock adapts automatically.

The slice size is a deliberate compromise. Too small and the overhead of yielding and resuming dominates, and the whole build takes noticeably longer in wall-clock for no responsiveness a reader can feel. Too large and a slice starts brushing the 50 ms line again.

Around 24 ms sits under the threshold with headroom for the browser's own frame work in the gap.

I also checked the thing time-slicing makes easy to get wrong: I diffed the sliced build's output against the old synchronous one and confirmed it was byte-identical. On this data a subtle reordering could change a count.

**A fast wrong number is worse than a slow right one.**

Proving I had changed only the timing and not the result was not optional.

Result: blocking time on the explorer dropped from about 795 ms to about 148, a 81 percent reduction, measured at 4x CPU throttle.

I want to be precise about that measurement's limit. It was taken with a long-task proxy, not the same Lighthouse harness that produced the 77-to-42 figure, so a clean confirming Lighthouse run on the fixed build is work I have not done. The relative drop holds; the harness match does not.

### A different tool for the same scarce resource

The dashboard and profile-listing pages had a related but distinct problem: they fetched large grouped lists and probed ~3,300 images on load, all up front, most of it below the fold.

I deferred it with an IntersectionObserver, the browser API that cheaply reports when an element is near the viewport without the jank of manual scroll listeners.

Three details make the difference between a deferral that helps and one that users feel as lag:

```ts
// 1. Degrade to eager loading rather than never loading.
if (typeof IntersectionObserver === 'undefined') return probeNow();

const observer = new IntersectionObserver(
  (entries) => {
    if (entries[0]?.isIntersecting) {
      observer.disconnect();  // 2. one-shot: never re-fire
      probeNow();
    }
  },
  { rootMargin: '200px' },    // 3. fire *before* it is visible
);
observer.observe(node);
```

**Fire early.** The avatar probes use a 200 px `rootMargin` and the below-the-fold list fetches use 400 px, so work starts before the element is actually on screen and the data is ready by the time the reader arrives. Deferral without a margin trades a slow first load for visible pop-in, which readers experience as a worse bug than the one you fixed.

**Disconnect after firing.** Otherwise the observer keeps calling back on every scroll past.

**Fail open, not closed.** If the API is missing or no element ever mounts, it loads eagerly. The fallback path is the old behaviour, so the worst case of this optimisation is the performance we already had.

Initial-load data on the dashboard dropped from ~26 MB to 0.07 MB, and the profile listing from ~11 MB to 0.09 MB. On-load requests dropped from thousands to under a hundred on both.

Time-slicing and deferral are two disciplines for protecting the same scarce resource. Neither deletes work. One spreads an unavoidable computation so it never blocks; the other declines to do off-screen work until it is warranted.

### The part of that win I will not overstate

That is a time-to-interactive win, not a total-bytes win.

A reader who scrolls all the way down still transfers the full grouped lists, around 26 MB on the dashboard, just lazily and without blocking the first view.

The bytes only disappear when the 5x redundancy from Part 11 is removed at the pipeline. I scoped it. I did not do it. I am not going to describe the page as fixed when a full scroll still moves tens of megabytes.

Deferral at least does not make the main-thread problem worse, which the browser-side de-duplication alternative would have.

The through-line of this whole part:

**Optimization moves the bottleneck. It rarely deletes it.**

I moved this system's cost from the network, where it made the site unusable for a bandwidth-constrained audience, onto the CPU, where it shows up in lab scores and on low-end devices. For this audience that is the right trade, because a page interactive in seconds that then does CPU work beats a page still downloading ten minutes later.

But it is a trade, not a free win. The explorer's main-thread cost is largely paid down. The dashboard's is not.

---

## Part 16: What I did not build

Most of the decisions above were decisions not to build something. They are scattered through the narrative because that is the order they happened in, so here they are in one place, which is also the form I would want to hand to whoever picks this up next.

Counting it up, the work that shipped is smaller than the work that was proposed and declined. That ratio is deliberate, and it is the part of the job I would most want judged.

| Proposed | Declined because | Would become right when |
|---|---|---|
| Framework migration as the performance fix | The bottleneck was 149 MB on the wire; the framework renders it no faster | Never, for this purpose. Done later as maintainability work on its own merits |
| Changelog-based incremental sync | Deletes are invisible to polling; webhooks add surface to the one thing kept unreachable | The source grows a real deletion event and the pipeline becomes the bottleneck |
| Query engine in the browser (WASM) | Engine binary rivals the whole encoded dataset; cold start lands worst on low-end phones | ~10x data growth, where an in-memory filter stops being obvious |
| Row-level delta sync | Version chains, compaction, stale-client rebasing, for a repeat-visit win immutable chunks already deliver | Repeat-visit bytes become the dominant cost, which caching currently prevents |
| Migrating to a zero-egress object store | Rewriting hardcoded URLs across two codebases for a latency gain small next to the payload win | Traffic grows into real egress bills |
| Table virtualization | Profiling showed ~20 DOM rows already; it solved a problem the table did not have | The table stops paginating |
| Web Worker for the filter build | Serialising across the boundary and restructuring the explorer, when slicing fixed it | Residual main-thread cost after slicing becomes the dominant complaint |
| Browser-side de-duplication of the 5x lists | Adds main-thread work to a system whose bottleneck had just moved to the main thread | Never; the fix belongs in the pipeline |

Two of those I was ready to start writing before I measured: virtualization, and the framework migration.

Every row has a named condition in the third column. That is the part I would insist on: **a rejection without a trigger is an opinion, and opinions expire silently.** A rejection with a trigger is a decision the next person can re-evaluate on evidence instead of re-litigating from scratch, which is the difference between leaving a system and leaving a system someone can operate.

One row is missing from that table, because it is a trigger on the constraint the whole design rests on rather than on any single decision: what happens when the data stops updating once a day. Hourly would not break this, and working out why was more useful than the answer. Publish frequency is not the load-bearing property. What matters is how much of the dataset changes per period, because content-defined chunking only re-publishes the chunks whose contents moved. At twenty to twenty-five new records a day, an hourly rebuild would change a handful of chunks and leave the rest of a reader's cache valid, so going twenty-four times more often costs close to what going once costs.

It fails on a different axis. When a reader is no longer allowed to be as stale as the publish interval, per-minute freshness or a correction that has to reach people before they act on it, a read path goes back in front of the data and Part 2 has to be re-costed from the top. Frequency was cheap. Bounded staleness is what the architecture actually bought, and it is the thing to test before reusing any of this.

## Where this actually stands

The work is through review and going to production as a single cutover with the migration, rather than a piecemeal swap that would leave the pipeline and the frontend disagreeing about the data shape mid-flight. Given that the contract between them is the thing holding this together, staging that switch in two halves would be the one obviously bad way to ship it.

The numbers above were measured on a develop deployment against a test bucket and distribution. I am not going to relabel them as production figures because the deploy is imminent, so here is exactly what changes and what does not when it lands.

**What will not change:** the byte counts. Payload is a property of what the pipeline writes, not of where it is served from, and this was verified rather than assumed: the same dataset measured 5.16 MB through the CDN against 5.18 direct from storage, at 45 of 45 edge hits. The compression, chunking and caching wins travel unchanged.

**What should improve:** the dashboard and listing figures, in both directions. The dev bucket has no profile images, so roughly 3,300 avatar requests fail there and inflate the wall-clock tail. Production has the images, cached, plus the lazy-loading fix that takes those requests to about thirty on load.

**What is still unproven:** the render scores. The first batch was quarantined for mixed provenance, and the time-slicing win was measured with a long-task proxy at 4x throttle rather than the harness that produced the original regression. The relative drop is solid; a clean confirming run on the shipped build is genuinely outstanding work, not a formality I am waving through.

**What ships with the cutover:** the counts fix from Part 13. It has been corrected in the pipeline since the work was done and reaches readers with this deploy. The incorrect output is archived as evidence rather than quietly overwritten, which is the only version of that decision I would defend.

Two limits hold regardless of deployment. The dashboard and listing "after" totals compare JSON components, not images. And every cross-audience figure is a projection from measured bytes divided by stated bandwidth, labelled as such wherever it appears, because I have instrumented captures and one real-user observation, not a fleet of throttled devices.

The first thing I want after cutover is not a number from this article. It is a threshold on the client-weight cost from Part 4, and something that pages someone when it is crossed.

What is next is the same discipline applied one more turn. The pipeline already knows which records changed on each run, so it can publish deltas against a base snapshot and let the client keep its copy in a local store, taking repeat visits from megabytes to kilobytes. The 5x grouped-list redundancy collapses into the index the explorer already caches, which would take those pages from tens of megabytes toward the 2.6 MB the index already costs. The remaining main-thread work there comes off the critical path the way the explorer's did.

One item on that list is not about performance. The provenance gap from Part 1 is still open: a reader verifies nothing, and has had no way to for four years. It was not rejected, which would at least have been a decision on record. It was never on the table, because we were busy counting the services we had removed and never asked what secured the one we kept.

Closing it means the pipeline signing what it publishes, the client refusing a manifest whose signature does not verify, and a public key served from somewhere other than the bucket being verified. The mechanism is well understood and the work is bounded. What it needs is someone to own a key, which is precisely the kind of standing obligation this architecture was built to avoid. That is the honest reason it is still open, and it is not a good enough one.

And the browser query engine stays on the shelf with its trigger written next to it: roughly a tenfold growth in the data, at which point an in-memory filter stops being the obvious choice.

---

## What I would take to the next system

None of the techniques here are inventions. Content-defined chunking comes out of deduplication and backup systems, yielding to the event loop between slices is a documented browser pattern, and the network-idle trap has been written up by people who found it before I did. I knew the names of most of them.

Knowing the names is not what stopped me reaching for virtualization first, or trusting a capture tool that was wrong by two orders of magnitude, or assuming a 570 MB page meant a large dataset. The techniques were available the whole time. What was missing each time was measuring before choosing, and that has to be applied on the day rather than remembered once.

So the five things below are the checks I would run earlier next time.

Five things, in the order they would change a decision.

**Let the threat model and the update frequency pick the architecture, before the framework conversation starts.** Those two properties eliminate more options, faster, than any preference about tooling. A daily update cycle and an operator who could be targeted narrowed three plausible architectures to one before anyone opened an editor.

The check that does the work is one question:

**How stale is a reader allowed to be?**

If the honest answer is hours rather than milliseconds, you probably do not need most of the infrastructure you were about to build, and every piece you skip is a piece nobody has to defend, patch, or pay for.

**Measure the wire before you touch the framework, and distrust the instrument before you distrust your understanding.** This project produced two separate cases of the tool lying: a network-idle heuristic reporting ~149 MB as 0.79 MB, and HEAD requests showing a CDN as permanently cache-missing. Both would have produced confident, wrong write-ups. A control page and a quarantine folder are cheap insurance against publishing either.

**Decompose before you optimise.** The dashboard's ~570 MB looked like a big-data problem and was a data-modelling problem: one dataset stored five times. The "avatar storm" looked like a byte problem and was a request-count problem: of ~570 MB, only about 4 MB was images. Both assumptions survived until something split the total by type, and both would have sent me at the wrong fix.

**Design the seams, and design for whoever inherits them.** The parts of this system I am most confident in are not the fast parts. They are the versioned contract between two repositories that share no code, the client that is allowed to refuse an index it does not understand, the pipeline that stops instead of publishing a smaller number, and the fallback I deliberately broke before trusting. None of those made anything faster. All of them make it harder for the next person to break this quietly, which on a platform like this matters more than milliseconds.

The corollary is why the simpler option kept winning. It was never an aesthetic preference. A system maintained by one or two people has a hard ceiling on the cleverness it can carry, and building above that ceiling hands someone a bill they never agreed to.

**A tradeoff you have accepted still needs a budget and an alarm.** This is the one I got wrong, and it is why this article has two halves.

I want to be exact rather than flattering, because the distinction is the lesson. The client-weight cost was not a surprise: it is the direct, obvious consequence of moving work out of the request path, and we accepted it knowingly. What I cannot claim is that we wrote it down with a number attached, because no design-era document I can produce says so.

That is the actual failure, and it is worse than forgetting. Understanding a tradeoff and instrumenting it are different acts, and only the second one outlives the people who made the decision. We had the understanding. We left no threshold, no budget, and no alert, so the cost grew for years with nobody assigned to notice until it was measured in minutes.

A tradeoff nobody is watching is just a bill you have not opened yet.