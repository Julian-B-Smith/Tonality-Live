# q020-conform-bound — a correction we did not need, and a measurement that changed what to build

Inbound: `tonality-live-conform-bound` (Tonality, 2026-08-15, from their audit
issue #262). They corrected a promise they had made us — that a conform snap can
never exceed 6 semitones "by construction" — which is false at the MIDI boundary,
where the nearer scale tone can fall outside 0..127 and the in-range one taken
instead may be further. They set `ball: consumer` because one of our pinned
contract tests asserts the false version.

## The premise about us was wrong

`git grep` over this tree finds `≤ 6` in exactly one place: `traces/2026-07-13-
ratify-q003.md`, recording "max snap distance observed = 6 (their ≤ 6 guarantee
holds)". That is an observation of a sample, not an assertion, and a trace is
history — not a gate. `./verify`'s `/transform` contract check asserts note
count, `NoteDescription` shape, and the presence of `collisions` /
`notes_snapped` / `ties_resolved` / `tie_break`. No delta bound anywhere.

So there was nothing to relax. The three contracts were *proposed by us and
landed in the provider's CI* (consumer-authored, resident-landed — INTEGRATIONS
§the contract test suite), so the stale assertion is in their tree. The reply
says so and asks them to check.

Worth naming because the pull was the other way: adopting a correction is
agreeable, cheap, and looks responsive. Adopting one we did not need would have
left both sides believing a fix had happened where none had.

## Measurement changed what was worth building

They reasoned the false claim "cannot fail for you — catalog max gap is 4". True
in direction, but I swept it rather than accepting it: every catalog scale × every
root × the boundary pitches, through the live bridge.

| | |
|---|---|
| edits examined | 4,914 (37 scales × 12 roots × 26 pitches) |
| carrying `tie_resolution: "range"` | 472 (~10%) |
| worst `abs(delta)` with the flag | **3** — Hirajoshi, root 1, 127 → 124 |
| worst without it, in that set | 2 |
| snaps outside 0..127 | 0 |

Two findings, the second of which is the useful one:

1. The corrected invariant holds here with a wide margin — 3, not 6.
2. **`"range"` is common, and does not mean what its name suggests.** 472 hits,
   nearly all 1-semitone moves. It marks *the register decided which side of a
   tie won*, not *this note jumped*. Their notice implies the latter.

That reframing is what settled the build question. They suggested surfacing
`"range"` in the UI the way collisions are surfaced, since "a large unexplained
jump reads as a bug even when it is the only legal answer". We do draw moved
notes against their original position — but the largest reachable jump is 3
semitones, which nobody reads as a bug. So the callout would be permanent UI
weight guarding a condition that cannot occur. Declined on grounds, recorded as
a grounded no rather than quietly skipped.

## What was built instead

`./verify full` now pins the invariant the provider *does* guarantee, against a
MIDI-boundary fixture (human-approved — editing the oracle is a §Domain gate):

- `0 <= to_midi <= 127`, always
- `abs(delta) <= 6 or tie_resolution == "range"` — the real bound
- the edit list is non-empty

The third is not padding. The fixture works because `Ionian` rooted at pc 11
straddles the edge; if a catalog change made it stop producing edits, the other
two assertions would pass vacuously and the check would read green while testing
nothing. That failure mode — a gate that quietly stops exercising its case — is
the same one the kit vendoring work turned up two days ago in a different guise.

**Each assertion was proved to fire**, since a gate never seen to fail is not
known to be a gate:

| crafted case | result |
|---|---|
| `delta 11`, `tie_resolution: "range"` (their worked case) | passes — correct, the flag licenses it |
| `delta 11`, `tie_resolution: "previous"` | **caught** |
| pitch 128 | **caught** |
| empty edit list | **caught** |

## Evidence

- `./verify full` exit 0 against a live bridge (mts 0.1.0). New line:
  `/transform boundary -> 2 edits in range, 2 flagged tie_resolution=range, max |delta| 1`
- Reply filed at `<Tonality>/integrations/Tonality-Live/adopt-conform-bound.md`
  (mailbox exception, INTEGRATIONS rule zero) — uncommitted there; the provider's
  residents commit their own tree.

## Open

Offered to file the consumer-exposure sweep as a contract test in their CI, as a
regression check on catalog gap. Ball: none until they take it up.
