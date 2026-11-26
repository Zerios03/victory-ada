# Where the frame time goes, and what to do next

This is the plan this fork is worked to. Every number is from the test machine
(Windows 11, i7-14700KF, RTX 4070 12 GB, 32 GB RAM) unless it says otherwise,
and every estimate is marked as one until a run carries it. Numbers from other
forks are quoted as their authors reported them; nothing here was measured by
this fork on their hardware.

See [../diagnostics/README.md](../diagnostics/README.md) for the switch set the
runs use, and the README's status table for the frame lines being compared.

## The shape of a frame

In the first rooms: 110 ms a frame at about 4,700 draws, 86 to 102 ms on the
water at about 1,300 draws, 160 to 208 ms outdoors at 7,200 to 9,700 draws. Two
terms fit those points well: about 70 ms that does not depend on the draw count,
plus about 13 ms per 1,000 draws. The card idles 12 to 40 ms a frame, so the
thread that prepares and submits is the limit, not the card.

That fixed 70 ms is not one thing. The 295 s timing run (run 280) splits a
frame roughly into: copies 43 ms, the title's own lists 49.8 ms of card time
after `VICTORY_CARD_BUFFERS_MIN_KB=64`, the layer's send-and-hash lists 23 ms,
and the rest in waits, descriptor work and refresh steps (20,000 to 43,000 a
frame).

So the order below is by milliseconds available per line of code changed, not
by how interesting the problem is.

## 1. Turn on what is already written (no new code)

Seven switches are in the tree, off by default, unit-tested, and have never
been measured on a card other than the one that wrote them. Measuring them is
a tester's afternoon and costs nothing but log files:

| Switch | What it should save | Confidence |
|--------|--------------------|------------|
| `VICTORY_STEP_INDEX=1` | 1.5 to 4 ms a frame (a step's copy found by index instead of a lock and a search) | timed offline |
| `VICTORY_FILLS_PLAIN=1` with `VICTORY_COPY_PLAIN=1` | about 3 ms a frame (8,000 to 12,000 sub-256-byte fills copied in place) | timed offline |
| `VICTORY_LIGHT_REFRESH=1` with `VICTORY_ABSENT_ONCE=1 VICTORY_ABSENT_MS=told` | the copies line 43 -> about 30 ms in rooms, 80 -> about 57 ms outdoors | estimate |
| `VICTORY_ASYNC_SUBMIT=1` | moves submission off the frame thread | unmeasured here |
| `VICTORY_JOIN_PARTS=1` | fewer, larger sends | unmeasured here |
| `VICTORY_CARD_BUFFERS_MIN_KB=64` with `VICTORY_CARD_BUFFERS_MB=1280` | 68.0 -> 49.8 ms of list card time (measured); the frame 114.9 -> 109.7 ms | measured |
| `VICTORY_CARD_HASH_LEN=1` | the 11.6 -> 23.0 ms the send-and-hash lists gained above | measured |

Each one needs two runs at the same place in the story, one with and one
without, and the frame line plus the `copies per frame` line from both.

## 2. The card loss (blocker)

Four losses in sessions of 6, 49, 31 and 38 minutes, all 0x887A0005, the last
three on the way to the same place in the story. No page fault address, no
driver event, video memory at 84 to 92 % of budget and never over it at a
sample, and nothing in the last seconds of the logs that is found only there.
`VICTORY_D3D12_DRED=faults` is cheap enough to leave on and is in the tree.

This comes before any further speed work, for two reasons: thirty minutes is
the longest a measurement can run, and a session that dies on the way to the
next save point cannot compare two builds at all. The 64 KiB constant view cap
(already on, pull request #911) is the one known invalid call another tester's
NVIDIA card answered with a removal; whether it is this loss is not known.

What a tester should send: `crashes.log`, the frame line before the loss, and
the memory lines. What would settle it fastest is a card with more memory: if
the loss moves with the card, it is memory; if it stays at the same place in
the story, it is what is drawn there.

## 3. Port the AMD fork's submission split, then measure it

[avatarmanz/VIctory-7900XTX](https://github.com/avatarmanz/VIctory-7900XTX)
reports 280 ms -> 50 to 75 ms frames in two days on an i9-12900K, 64 GB RAM,
RX 7900 XTX, with 40 to 46 GB of process memory. Its README and commits name
the changes: a thin command stream (`VICTORY_THIN`), per-step card-side work
(`VICTORY_CARD_STEPS`), a landing thread (`VICTORY_LAND`), deferred lists, the
copy queue used for texture uploads, and cheaper write traps. This fork already
has the related `VICTORY_ASYNC_SUBMIT`, `VICTORY_RELAY` and `VICTORY_LATE_WAIT`,
which is why the card's idle time behind the submitting thread already fell
from about 136 to 30 ms a frame in a first run.

The remaining gap, and what to do about it:

- **A thin stream.** The layer's own lists cost 11.6 to 23.0 ms of card time a
  frame; most of that is hashing whole 64 KiB blocks for small copies. A stream
  that carries only what changed, with the hash taken over the copy's own bytes
  (`VICTORY_CARD_HASH_LEN=1` is the first step), is the same idea and is
  cheaper to check.
- **Copy-queue uploads.** Texture uploads share the direct queue with draws.
  Moving them to the copy queue cannot be verified in this tree without a card,
  so this waits for a tester on the build that has it.
- **Cheaper write traps.** The guard over every view of a memory page came from
  myokex/victory-amd. Its per-write cost is not in any profile this fork has;
  measuring it is a `VICTORY_GPU_TIMING=1` run away.

What this fork does not do is copy their patches. Every one of those changes is
a change to this tree's own structures, and the ones that are already here were
written against this tree and are measured against its frame line.

## 4. Shader work, last

The RDNA3 fork measured its float-shortcut work at 18.41 -> 17.58 ms of
dispatch and draw, 0.4 ms of a 36.6 ms frame. The clean-up passes here are
worth more (about 43 % fewer host instructions, about 20 ms of card time), but
that is already in. Further shader work is a legitimacy and correctness
programme (the 118 refusals in `crates/victory-shader/src/emit/lower.rs`, the
black shadowed areas, the ray-tracing builder's last 5 programs) rather than a
frame-rate one, and it should not be sold as one.

## How a change is accepted

1. Unit test or fixture, in the tree, that fails before and passes after.
2. `cargo clippy --workspace --all-targets --locked -- -D warnings` and
   `cargo test --workspace --locked`, which is what CI runs.
3. A switch, off by default, if the change touches a path that might be wrong
   on another vendor's card.
4. Two runs from a tester, at the same place in the story, with and without.
5. Only then a line in the README's status table, with the run number.

Estimates that no run has carried are marked as estimates in the README and
here. That rule is the point of the fork; it is also what makes it useful to
somebody who is not its author.
