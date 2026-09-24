# Designing and Maintaining a Civic Platform Under Constraints

## Off the wire, onto the device

_The second half of a two-part case study. [The first half](../README.md) is the design and the network: the threat model that ruled out a public API, the bill that arrived years later, and getting 932 MB of cold-visit weight down to about 5. This half is what happened once the bytes were no longer the problem. The work moved off the wire and onto the device, and the device is a low-end phone._

---

## Part 13: The migration, and the bugs a compiler found

With the delivery path fixed, the migration I had deferred was worth doing, now honestly framed as maintainability work rather than a performance fix.

I chose a clean rebuild on the App Router rather than an in-place 14 to 15 to 16 upgrade. The old code health made the rebuild cheaper than the upgrade path, which is not the usual answer and was the right one here.

Next.js 16, React 19, strict TypeScript, every route ported behaviour for behaviour. Two details would catch anyone doing the same move on Next.js 16: the internationalisation library we were on does not support the App Router, which meant rewriting every message call site, and the middleware entry point is renamed.

Routing behaviour, including localised URLs, stayed identical, because changing user-visible URLs on a site other people link to is its own kind of breakage.

The change that paid for the whole migration was **turning type checking on.** The old build had `ignoreBuildErrors` set, which is how a years-old codebase ends up shipping bugs a compiler would have caught on the first build.

The clearest example: the explorer's Reset button threw a `ReferenceError` on every single click. It called a setter that never existed. With type checking off the build never complained, and because the error only fired on click, it survived in production for years.

### Three pipeline defects worth naming plainly

These are the kind that hide in a system that reports success too easily.

**The pipeline could publish a truncated dataset and report success.** If a page of the source fetch failed after its retries, the code caught the error, logged it, and carried on, and the top-level handler exited zero. A partial dataset would publish over the good one, with undercounted numbers, and the scheduled job would go green.

On a platform whose figures other people rely on, silently shipping a smaller number is the worst class of bug available to it.

The fix was to fail loudly: throw when the fetched row count does not match the source's authoritative total, and set a non-zero exit code in the top-level handler so the CI chain halts and the publish step never runs on a bad fetch.

A pipeline that stops is safe. A pipeline that continues past an error and publishes anyway is not.

**The second defect had already reached production.** One side of a two-way status split was publishing as zero, so an entire category was presented under the wrong label.

The cause was one level of unwrapping. The source stores certain fields as a small object with the value nested inside, and the pipeline read one level too shallow, comparing an object against a string that never matched.

The reason it went unnoticed is the instructive part. The frontend read the same field correctly, because its own parser happened to unwrap twice. The interactive table showed the right thing while the summary count showed zero.

Two code paths reading the same source with different assumptions is a bug waiting for one of them to be right by accident.

**The third was a security defect.** The pipeline authenticated to the source with a token and followed the source's pagination links verbatim. Those links came back over plain HTTP even though the base URL was HTTPS.

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
| Listing page | ~209 MB | ~13 MB | −93.9% |
| All four pages | 932 MB | 49 MB | −94.8% |

Explorer time-to-loaded on the same machine and network went from 42 seconds to about 6, with first contentful paint at 244 ms. Projected against reader bandwidth, first-visit table data goes from ~13.3 minutes to ~14 seconds at 1.5 Mbps, and from ~50 minutes to ~53 seconds at 400 kbps.

Those are instrument numbers, and instrument numbers are not the point of this system. The one that matters came from someone opening the dashboard on their own machine and their own connection, without a capture tool involved: **about four to five minutes minimum, down to about forty-five seconds.**

I checked it against the instrumented values rather than just accepting the good news, because a pleasing anecdote is the easiest thing in the world to publish. It reconciles: the old dashboard measured 4 m 40 s at 570 MB, and the new one at ~28 MB is roughly 14 seconds of transfer at that bandwidth. The remaining thirty seconds is almost entirely an artifact of the dev environment, where thousands of image requests fail against a bucket that has none of those images. On production, where those images exist and cache, and with the lazy-loading fix taking the request count from thousands to about thirty, that tail should mostly disappear.

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

The explorer had one bad offender: a single synchronous function building the crossfilter index over every row in roughly 747 milliseconds of uninterrupted work.

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

Result: blocking time on the explorer dropped from about 795 ms to about 148, an 81 percent reduction, measured at 4x CPU throttle.

I want to be precise about that measurement's limit. It was taken with a long-task proxy, not the same Lighthouse harness that produced the 77-to-42 figure, so a clean confirming Lighthouse run on the fixed build is work I have not done. The relative drop holds; the harness match does not.

### A different tool for the same scarce resource

The dashboard and listing pages had a related but distinct problem: they fetched large grouped lists and probed thousands of images on load, all up front, most of it below the fold.

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

**Fire early.** The image probes use a 200 px `rootMargin` and the below-the-fold list fetches use 400 px, so work starts before the element is actually on screen and the data is ready by the time the reader arrives. Deferral without a margin trades a slow first load for visible pop-in, which readers experience as a worse bug than the one you fixed.

**Disconnect after firing.** Otherwise the observer keeps calling back on every scroll past.

**Fail open, not closed.** If the API is missing or no element ever mounts, it loads eagerly. The fallback path is the old behaviour, so the worst case of this optimisation is the performance we already had.

Initial-load data on the dashboard dropped from ~26 MB to 0.07 MB, and the listing page from ~11 MB to 0.09 MB. On-load requests dropped from thousands to under a hundred on both.

Time-slicing and deferral are two disciplines for protecting the same scarce resource. Neither deletes work. One spreads an unavoidable computation so it never blocks; the other declines to do off-screen work until it is warranted.

### The part of that win I will not overstate

That is a time-to-interactive win, not a total-bytes win.

A reader who scrolls all the way down still transfers the full grouped lists, around 26 MB on the dashboard, just lazily and without blocking the first view.

The bytes only disappear when the 5x redundancy from [Part 11](../README.md#part-11-the-570-mb-that-was-not-a-big-dataset) is removed at the pipeline. I scoped it. I did not do it. I am not going to describe the page as fixed when a full scroll still moves tens of megabytes.

Deferral at least does not make the main-thread problem worse, which the browser-side de-duplication alternative would have.

The through-line of this whole part:

**Optimization moves the bottleneck. It rarely deletes it.**

I moved this system's cost from the network, where it made the site unusable for a bandwidth-constrained audience, onto the CPU, where it shows up in lab scores and on low-end devices. For this audience that is the right trade, because a page interactive in seconds that then does CPU work beats a page still downloading minutes later.

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

One row is missing from that table, because it is a trigger on the constraint the whole design rests on rather than on any single decision: what happens when the data stops updating once a day. Hourly would not break this, and working out why was more useful than the answer. Publish frequency is not the load-bearing property. What matters is how much of the dataset changes per period, because content-defined chunking only re-publishes the chunks whose contents moved. At the current daily intake, which is a tiny fraction of the dataset, an hourly rebuild would change a handful of chunks and leave the rest of a reader's cache valid, so going twenty-four times more often costs close to what going once costs.

It fails on a different axis. When a reader is no longer allowed to be as stale as the publish interval, per-minute freshness or a correction that has to reach people before they act on it, a read path goes back in front of the data and [Part 2](../README.md#part-2-three-options-costed-before-any-code) has to be re-costed from the top. Frequency was cheap. Bounded staleness is what the architecture actually bought, and it is the thing to test before reusing any of this.

## Where this actually stands

The work is through review and going to production as a single cutover with the migration, rather than a piecemeal swap that would leave the pipeline and the frontend disagreeing about the data shape mid-flight. Given that the contract between them is the thing holding this together, staging that switch in two halves would be the one obviously bad way to ship it.

The numbers above were measured on a develop deployment against a test bucket and distribution. I am not going to relabel them as production figures because the deploy is imminent, so here is exactly what changes and what does not when it lands.

**What will not change:** the byte counts. Payload is a property of what the pipeline writes, not of where it is served from, and this was verified rather than assumed: the same dataset measured 5.16 MB through the CDN against 5.18 direct from storage, at 45 of 45 edge hits. The compression, chunking and caching wins travel unchanged.

**What should improve:** the dashboard and listing figures, in both directions. The dev bucket has none of those images, so thousands of image requests fail there and inflate the wall-clock tail. Production has the images, cached, plus the lazy-loading fix that takes those requests to about thirty on load.

**What is still unproven:** the render scores. The first batch was quarantined for mixed provenance, and the time-slicing win was measured with a long-task proxy at 4x throttle rather than the harness that produced the original regression. The relative drop is solid; a clean confirming run on the shipped build is genuinely outstanding work, not a formality I am waving through.

**What ships with the cutover:** the counts fix from Part 13. It has been corrected in the pipeline since the work was done and reaches readers with this deploy. The incorrect output is archived as evidence rather than quietly overwritten, which is the only version of that decision I would defend.

Two limits hold regardless of deployment. The dashboard and listing "after" totals compare JSON components, not images. And every cross-audience figure is a projection from measured bytes divided by stated bandwidth, labelled as such wherever it appears, because I have instrumented captures and one real-user observation, not a fleet of throttled devices.

The first thing I want after cutover is not a number from this article. It is a threshold on the client-weight cost from [Part 4](../README.md#part-4-what-we-accepted-in-exchange), and something that pages someone when it is crossed.

What is next is the same discipline applied one more turn. The pipeline already knows which records changed on each run, so it can publish deltas against a base snapshot and let the client keep its copy in a local store, taking repeat visits from megabytes to kilobytes. The 5x grouped-list redundancy collapses into the index the explorer already caches, which would take those pages from tens of megabytes toward the 2.6 MB the index already costs. The remaining main-thread work there comes off the critical path the way the explorer's did.

One item on that list is not about performance. The provenance gap from [Part 1](../README.md#part-1-the-box-we-were-designing-in) belongs on it. It was not rejected, which would at least have been a decision on record. It was never on the table, because we were busy counting the services we had removed and never asked what secured the one we kept.

Closing it means the pipeline signing what it publishes, the client refusing a manifest whose signature does not verify, and a public key served from somewhere other than the bucket being verified. The mechanism is well understood and the work is bounded. What it needs is someone to own a key, which is precisely the kind of standing obligation this architecture was built to avoid. That is not a good enough reason to leave it.

And the browser query engine stays on the shelf with its trigger written next to it: roughly a tenfold growth in the data, at which point an in-memory filter stops being the obvious choice.

---

## What I would take to the next system

None of the techniques here are inventions. Content-defined chunking comes out of deduplication and backup systems, yielding to the event loop between slices is a documented browser pattern, and the network-idle trap has been written up by people who found it before I did. I knew the names of most of them.

Knowing the names is not what stopped me reaching for virtualization first, or trusting a capture tool that was wrong by two orders of magnitude, or assuming a 570 MB page meant a large dataset. The techniques were available the whole time. What was missing each time was measuring before choosing, and that has to be applied on the day rather than remembered once.

So the five things below are the checks I would run earlier next time.

Five things, in the order they would change a decision.

**The threat model and the update frequency decided more here than the framework ever could.** Those two properties eliminate more options, faster, than any preference about tooling. A daily update cycle and an operator who could be targeted narrowed three plausible architectures to one before anyone opened an editor.

The check that does the work is one question:

**How stale is a reader allowed to be?**

If the honest answer is hours rather than milliseconds, most of that infrastructure is not needed, and every piece skipped is a piece nobody has to defend, patch, or pay for.

**The tool lied twice, and both lies were convincing enough to publish.** This project produced two separate cases of the tool lying: a network-idle heuristic reporting ~149 MB as 0.79 MB, and HEAD requests showing a CDN as permanently cache-missing. Both would have produced confident, wrong write-ups. A control page and a quarantine folder are cheap insurance against publishing either.

**A total says nothing about its own shape.** The dashboard's ~570 MB looked like a big-data problem and was a data-modelling problem: one dataset stored five times. The "image storm" looked like a byte problem and was a request-count problem: of ~570 MB, only about 4 MB was images. Both assumptions survived until something split the total by type, and both would have sent me at the wrong fix.

**The seams are the part I would defend, not the speed.** The parts of this system I am most confident in are not the fast parts. They are the versioned contract between two repositories that share no code, the client that is allowed to refuse an index it does not understand, the pipeline that stops instead of publishing a smaller number, and the fallback I deliberately broke before trusting. None of those made anything faster. All of them make it harder for the next person to break this quietly, which on a platform like this matters more than milliseconds.

The corollary is why the simpler option kept winning. It was never an aesthetic preference. Every system has a ceiling on the cleverness its maintainers can carry, and building above that ceiling hands someone a bill they never agreed to.

**An accepted tradeoff still needs a budget and an alarm.** This is the one I got wrong, and it is why this article has two halves.

I want to be exact rather than flattering, because the distinction is the lesson. The client-weight cost was not a surprise: it is the direct, obvious consequence of moving work out of the request path, and we accepted it knowingly. What I cannot claim is that we wrote it down with a number attached, because no design-era document I can produce says so.

That is the actual failure, and it is worse than forgetting. Understanding a tradeoff and instrumenting it are different acts, and only the second one outlives the people who made the decision. We had the understanding. We left no threshold, no budget, and no alert, so the cost grew for years with nobody assigned to notice until it was measured in minutes.

A tradeoff nobody is watching is just a bill you have not opened yet.