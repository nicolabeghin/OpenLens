# Chat transcript hang: scene-update watchdog kill (`0x8BADF00D`)

Branch: `fix/chat-scroll-repin-loop` · Found: 2026-11-05, while verifying the sessions filter on a physical iPhone.

**Status: the loop is confirmed; its driver is not.** The leading theory (a scroll repin feedback
loop) fits every observation but has not been isolated. The plan below is built to separate it from
its rivals before any fix is trusted.

## Symptom

After opening a few sessions the app froze ("stuck"), then restarted. Reported as a suspected
memory leak across viewed sessions.

## It is not memory

| Evidence | Reading |
|---|---|
| Crash report: `EXC_CRASH` / `SIGKILL`, namespace `FRONTBOARD`, code `0x8BADF00D` | A **watchdog** kill. A jetsam kill reads `EXC_RESOURCE` or appears only as a `JetsamEvent`. |
| `scene-update watchdog transgression … exhausted real (wall clock) time allowance of 10.00 seconds` | The main thread did not finish one scene update in 10 s. |
| `Elapsed application CPU time 9.996 s`, 10.00 s of wall clock | It was **computing the whole time**, not blocked on a lock or the network. |
| `vmSummary`: Malloc 97 MB, writable regions 182 MB | Unremarkable. Nothing was starved. |
| Separate `JetsamEvent` (10:11, a different launch, pid 8250): OpenLens *suspended* at 4,317 pages (~69 MB), lifetime max ~165 MB. Largest process was Gmail. | OpenLens was a bystander in a system-wide snapshot, not the victim. |

Also checked in code and **ruled out** as growth paths: per-session caches in `ChatClient` (only
`draftsBySessionID`, short strings), the event recorder (inert unless a capture is started),
cross-session event injection (`SSEEventHandler` guards on `currentSessionID`), the viewed-marker
hook (pure read, no write-back).

## Where the time went

Main-thread stack at the kill, innermost first:

```
DynamicBody.updateValue()                       <- a Button inside a body being re-evaluated
AG::Graph::UpdateStack::update()
GraphHost.flushTransactions()
ScrollViewCommitMutation.commit(layoutState:…)  <- a ScrollView's layout commit
… _UIHostingView.layoutSubviews()
```

No network or storage frames. The register state also holds `AGGraphCurrentAttributeWasModified`
(an attribute changed while the graph was updating). That is weak evidence on its own; the trace
below is what establishes a loop.

`ProcessVisibility: Background` and `WatchdogEvent: scene-update` suggest the freeze was already
running in the foreground, and the kill landed when the app was sent away and iOS asked the scene
to update. *Inferred, not shown.*

## The loop, measured

A DEBUG-only counter (`RenderTrace`, applied from the patch next to this file) counts `body`
evaluations per view and prints any view that runs 30+ times in a second. Launched on the phone
with the console bridged (`devicectl … --console`), **before any interaction**:

| View | Result |
|---|---|
| `ChatMessagesList` | 69 one-second windows over threshold, typically 190–227 renders/s, **peak 672/s** |
| `SessionsList` | never crossed the threshold |
| `ChatView` | never crossed the threshold |

Every sample showed identical inputs: `messages=286 content=4 timeline=13`. The view re-rendered
hundreds of times a second while nothing it reads was changing.

The existing `CHAT_STRESS_TIMELINE_REBUILD` log shows `displayed` and `items` **oscillating**
(15↔16, 24↔25↔26) as `version` climbs, and `version` sequences from more than one source
interleaving.

**Consequence for the sessions-list work:** the list is innocent. The user has 67 sessions, and
aggregating 67 sessions ten times per render is hundreds of microseconds. Memoising the aggregation
was proposed and **withdrawn**: it would cheapen wasted work without stopping it.

## Provenance

`ChatScrollPolicy.shouldRepinToBottom` arrived in `ec5aa6d` ("Keep long chats pinned to the latest
message"). That commit is in **upstream** `main` (`martynpekala/OpenLens` at `80c218c` contains two
occurrences), so this is a bug in the original project, not in the multi-server branch.

## Theories (none eliminated yet)

1. **Repin feedback loop (leading).** In `onScrollGeometryChange`'s action, `scrollState = state`
   then `shouldRepinToBottom` → `scrollToBottom`. The scroll changes the geometry, which re-runs
   the action. `shouldRepinToBottom` only requires `!isLoading && followLatest && allowsProgrammaticScroll
   && !isAtBottom && !hasPendingRestoration`. Its own doc comment names the case where it cannot
   settle: finished rows that keep growing after the jump. Its sibling `shouldAutoFollow` guards
   against this with `contentVersion != lastHandledContentVersion` and a `minimumInterval` check
   on `lastAutoScrollDate`; **the repin path writes `lastAutoScrollDate` and never reads it.**
2. **Layout flap, repin innocent.** `scrollState` is `@State` read by the body, so every geometry
   tick re-renders it. If lazy row measurement makes `contentSize` change continuously, the body
   spins at the layout rate with no repin involved. A repin guard would not cure this.
3. **Row growth** (Markdown finishing, the case in the doc comment) feeding either of the above.
4. **Several live transcripts.** The interleaved `version=` sequences may be more than one
   `ChatView` alive in a navigation stack, or several `ChatClient`s (every `ServerContext` builds
   one eagerly). Not established.
5. **Debug-build amplification.** The crashed process ran `OpenLens.debug.dylib`. A Debug build is
   slower, so Release may loop but survive, or not loop. Unknown.

## Open question for the multi-server branch

The loop was already running at **launch**, with a 286-message transcript on screen and no
interaction. Was that transcript restored by existing behaviour, or by this branch's per-server
"remember the last-open session"? If the latter, multi-server made an existing bug reachable on
every launch. To check: launch with a saved chat path on `main` before the merge and on this branch.

## Approach

**Principle: the loop is the thing to stop; a guard that cannot be shown to stop it is not a fix.**

1. **Reproduce and discriminate (needs the phone).** Re-apply the trace patch and add two counters:
   one on every entry of the geometry action, one on the branch that actually calls
   `scrollToBottom`. Reading: repin count ≈ geometry count ≈ body count means theory 1;
   geometry count high with repin count low means theory 2.
2. **Write the failing test first.** `ChatScrollPolicy` is a pure enum and
   `ChatStreamBehaviorTests` already covers its siblings. Express termination as a property:
   given a sequence of "not at bottom" observations with no user input, the policy must stop
   returning `true` after a bounded number of attempts, and must re-arm when the user scrolls, a
   new message arrives, or the bottom is reached.
3. **Fix the repin path with the guards its sibling has**, in this order of preference:
   a bounded attempt budget that resets on those three events (this is what guarantees
   termination); then pacing via the already-written `lastAutoScrollDate`. Pacing alone only turns
   an infinite loop into a slow one.
4. **Verify against the number, not the feel.** Same device, same transcript, trace on. Before:
   190–672 renders/s. After: a bounded burst at load, then silence. Also confirm "stays pinned to
   the latest message" still holds, since that is what #27 exists to provide.
5. **If theory 2 wins**, stop: a repin guard is then the wrong fix and the next step is why
   `contentSize` never settles.
6. **Check Release** (`scripts/run-device.sh --configuration Release`) so we know whether this
   ships to App Store users or is Debug-only.

## Definition of done

- A test that fails on `main` and passes on this branch.
- The trace shows `ChatMessagesList` under 30 renders/s, steady state, on the 286-message
  transcript that reproduced it.
- `./scripts/test-ios.sh` green, and `./scripts/verify-ui-parity.sh` unaffected.
- Trace code removed from the tree (it stays only as the `.patch`).

## Shipping

The bug is in upstream `main`, so the fix should be a **small standalone PR on upstream `main`**,
not part of the multi-server PR (#36). This branch is based on the merged fork `main`, so when it
is ready, cherry-pick the fix onto a branch cut from `80c218c`.

## Files

- `investigations/chat-scroll-repin-watchdog.render-trace.patch`: re-enables `RenderTrace` plus the
  three `body` hooks (`git apply`). Debug-only, inert unless launched with `OPENLENS_TRACE_RENDER`.
- Raw crash report and jetsam report are **not** committed (they carry device identifiers).
