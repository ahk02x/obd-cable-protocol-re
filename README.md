# I reverse-engineered my car's tuning cable

*Or: how a $600 Windows-only dongle became 150 lines of Python, and how I shipped a
bug that made a million readings quietly wrong.*

---

I drive a 2018 Audi SQ5 — B9, EA839 3.0T, Stage 2. Tuned cars get logged: you record
what the ECU is doing under load, and you look for the thing that's about to cost you
an engine.

The tool I have for that is a Jackal cable. It plugs into the OBD port, talks to the
engine ECU, and streams back 54 channels of internal ECU state — not the dozen public
OBD-II PIDs, but Bosch's own internal measurement variables: `nmot_w`,
`IKCtl_agKnkInt_as8[0..5]`, `zwoutzyl_w_msg[..]`. Per-cylinder knock retard. Per-cylinder
misfire counts. Rail pressure actual vs. requested. The stuff that actually tells you
something.

It has one problem: the software is Windows-only, and I work on a Mac.

I could have kept a Windows laptop in the passenger footwell. Instead I spent a couple
of weeks recovering the protocol, and now a Raspberry Pi in the trunk writes the same
CSV the vendor's app does — 54 columns, value-for-value.

Here's how, including the part where I was wrong for three weeks.

---

## Step 1: figure out what you're actually looking at

The cable enumerates on macOS as `/dev/cu.usbserial-0001`. The installer ships Silicon
Labs CP210x drivers. So the transport is a bog-standard USB-to-UART bridge — the
*hardware* is generic, the *protocol* is proprietary. That's a good starting position:
whatever's happening, it's bytes over a serial line, and I can see every one of them.

I opened the port and listened. Silence. Nothing unsolicited at idle, which told me the
dongle is request/response — it waits to be asked. So I couldn't just eavesdrop on a
stream; I'd have to learn the questions.

Next I went looking for a shortcut and didn't find one, which was itself informative.
The vendor app is a .NET MAUI desktop app from Automotive Data Technik AG. Its local
app-data directories are **empty**. It carries an `ADT.DocumentFormat.Odx.dll` — ODX
being ISO 22901, the standard format for diagnostic definitions — but no ODX files on
disk. Conclusion: the label-to-address map isn't cached locally, it's pulled from
their backend at runtime.

There was no map to lift. Which meant correlation was not the lazy path, it was the
only path.

## Step 2: capture a known-good session

The crux of the whole project. I put the cable on a Windows machine, installed USBPcap
and Wireshark, captured the USB root hub, and ran a deliberately boring, scripted
session:

> connect → ignition on → idle 15s → **one clean rev** → start log → idle → rev → idle
> → stop log → disconnect

And I saved two things: the pcap **and** the vendor's own CSV from that exact session.

That pairing is the whole trick. The pcap is bytes I don't understand. The CSV is
engineering units I do understand. They describe the same fifteen seconds. The rev is a
timing fiducial deliberately punched into both files — a big, unambiguous, monotonic
event I can find in either one.

I did it three times, so I could tell constant framing from changing data.

## Step 3: find the frame

Extract the CP210x bulk-transfer payloads from the pcap, concatenate, and stare.

The framing fell out fast — a repeating two-byte magic, then two big-endian length-ish
fields, then a payload:

```
bb 02 | CHK(2, BE) | LEN(2, BE) | SEQ(2) | payload[LEN]
```

The interesting field was `CHK`. I guessed the cheapest thing a small MCU would do —
a plain 16-bit sum of every byte in the frame except the checksum itself — and tested
it against the whole capture.

**285,832 frames. 100% match.** No CRC polynomial hunt, no lookup table. Just a sum.

That number matters more than it looks. A checksum that validates on a quarter-million
frames isn't a lucky guess on a hundred; it's a fact. From that point on I had a hard
oracle: any frame I built either checksummed or it didn't, and I could tell before I
ever sent a byte to my car.

Splitting by direction gave two shapes:

- **Requests** (host → dongle): a list of 5-byte entries, `[width][region(2)][address(2)]`
- **Responses** (dongle → host): a TLV list, `[len][value(len)]`, **in request order**

That last clause is the second gift. The response carries no identifiers at all. It's
positional. Whatever order you asked in, that's the order you get back — which means
the request frame is simultaneously the query *and* the schema.

The main request is 243 bytes and comes back as a 151-byte response about 150 ms later.
Widths are 1, 2, or 4 bytes; values little-endian; 4-byte fields are IEEE floats.

And — the answer to the question that decided whether any of this would work — there
was **no seed/key exchange.** No UDS `0x27`. Nothing session-dependent in the read path.
A blind replay would keep working tomorrow. (This is a diagnostic *read* path on my own
car; the write and flash services are a completely different story, and I don't touch
them. More on that below.)

## Step 4: correlate bytes to meaning

Now the fun part: I have raw values at known byte offsets, and a CSV with 54 named
columns. Align the two on the rev.

RPM is trivially first — it's the only channel that goes from ~800 to ~4000 and back in
two seconds. Find the response field that climbs in lockstep with the CSV's `nmot_w`,
and you've identified an address. Take two `(raw, physical)` pairs, solve
`phys = raw × scale + offset`, done.

Then coolant, intake air temp, throttle, pedal, load, rail pressure, boost actual and
requested, lambda both banks, air mass, gear. Then the per-cylinder blocks, which
announce themselves as six consecutive addresses moving together.

I wrote a script to do the fitting by correlation and let it grind through the capture
against the CSV. Out came a channel map: address → column name, unit, scale, offset.

Then I rebuilt the client from scratch in Python — replay the handshake, send the
request on cadence, parse the TLV response, apply the map, write a CSV with the
vendor's exact column headers.

I logged a drive with mine and diffed it against a real vendor log.

**54 columns. Value-for-value.**

Total: ~150 lines for the protocol layer, standard library plus `pyserial`. It runs on
the Mac; it runs headless on a Pi in the car. The corpus behind it is now 111 sessions
and 272,268 samples.

---

## The part where I was wrong for three weeks

I'd like to end there. Here's what actually happened next.

Two channels were silently decoded wrong, and I didn't notice, because they *looked*
fine.

The worst was per-cylinder knock retard — arguably the single most important thing in
the whole log, since it's the channel that tells you an engine is being damaged. Mine
had collapsed into two values: `0` and about `-0.80`. A binary flag. Knocking, or not.
No depth.

The root cause was in my own fitter. `build_channel_map.py` recovers
`value = raw × scale + offset` by linear correlation against the CSV — and **a fitted
line cannot express signedness.** When the true channel is a signed integer counting
down from zero, correlation instead smears the full unsigned 0–255 range across the
narrow output window it observed. You get numbers. They're in a plausible range. Every
row is populated. They mean nothing.

Worse, it was contaminated by *good driving*. The two captures I fitted from happened
to contain **no knock at all** — so the fitter drew a straight line through noise and
landed on a scale of `-0.0031 deg/count`, mapping all 256 raw values into a 0.8-degree
window.

Ground truth, once I went looking across 93,000 samples, was embarrassingly clean. The
value set is *exactly*:

```
{0, -0.75, -1.5, -2.25, -3.0, -3.75, -4.5}
```

That's a signed int8 at 0.75 deg/bit counting down from zero. Raw 255 is `-0.75`, raw
250 is `-4.5`. Not a fit. A quantum. It was sitting there in plain sight the whole time,
and my regression had averaged it into mush.

Two things came out of this that I'd argue are the real result of the project.

**First: my integrity checker was asking the wrong question.** It verified that every
channel was *present and populated*. Both broken channels were 100% present and 100%
populated. "Populated" and "correct" are not the same property, and I was only testing
the first one.

The fix is a liveness check that asserts a channel still *moves*: if it has more than
one distinct value but its total span sits below a physical floor, that's a fail, not a
pass — real engine behaviour never lives inside a sub-physical window. The floors sit
deliberately *between* the broken span and the true one (knock: broken 0.80, true 4.5,
floor 1.5). One distinct value only warns, because a knock-free week is legitimate and a
check that cries wolf gets ignored. It fails on all 12 broken channels and raises zero
false alarms across 12 known-good sessions.

**Second: the old data was recoverable, and knowing when to stop wasn't obvious.**

The broken transform was exactly invertible for knock, because at 0.0031 deg/count one
CSV rounding step is 0.3 raw counts — so `round((logged - offset) / scale)` gets the
original byte back, and `int8 × 0.75` gets the truth. That repaired **986,478 values
across 40 sessions**, zero strays, zero failed inversions. Real knock depth that had
been masquerading as a one-bit flag.

The detector for "is this session broken" had to be exact rather than heuristic. My
first version keyed on "cylinder 3's span is under one quantum" — which misfires on a
session where cylinder 3 simply stayed quiet, and would have rewritten that session's
genuine `-0.75` values on *other* cylinders into `-12.75`. The dry-run guard caught it
before it touched anything. The correct test is the presence of any **off-quantum**
value, since a correct decode can only ever emit multiples of 0.75.

And the second broken channel — per-cylinder ignition angle — **I did not fix, because
it isn't fixable from the logs.** Its fitted scale was ~0.00023 deg/count, so a single
CSV rounding step spans about five raw counts. The original bytes are gone. I tested the
obvious hypothesis (`int16 × 0.75`) against the recovered raws: only **45.9%** landed in
the physically plausible band, versus 100% for the same method on knock. So the
hypothesis is unsupported, and a confident-looking guess would have been strictly worse
than an honest gap. That channel is flagged `suspect: true`, its analysis window is
documented as closed, and recovering it needs fresh raw bytes at the car.

The rule I took away, which I now apply everywhere:

> **Fit unsigned magnitudes. Declare signed ones.** Any channel that is inherently
> signed — a retard, a correction, a delta — needs an explicit sign flag and a *known*
> quantum, never a correlation-fitted scale.

---

## What the data said about my car

Once the decode was actually right:

- **Cylinder 3 is the bad one.** Its misfire counter is the worst in **39 of 39 drives**
  — roughly 250–411 against 70–150 for its neighbours. Consistent enough that it's a
  hardware fault, not noise. Coil, plug, or injector. Next step is a coil swap to confirm
  the mapping moves with the part.
- **My knock is lugging knock, not heat.** It clusters at 2000–3000 rpm under high load.
  The high-rpm pulls — the ones that *feel* violent — stay clean. That is the opposite of
  what I assumed before I could see per-cylinder depth, and it changes how I drive.
- Boost peaks around 24–25 psi; air-mass estimate puts it near 428 hp.

That first one is the whole reason to do any of this. A generic OBD-II scanner would have
told me "misfire detected" or, more likely, nothing at all. Per-cylinder data over 39
drives told me *which* cylinder, *how consistently*, and *under what conditions* — before
it became a warning light.

---

## On the legality and the redactions

This is my car, my cable, my data. Reverse-engineering a device you own to make it
interoperate with your own computer is well-trodden ground, and vehicle diagnosis and
repair specifically sits inside the DMCA §1201 exemptions in the US, with comparable
interoperability provisions in Canada, where I am.

Two deliberate limits:

**Read-only, enforced in code.** The tool can send session control, tester-present, and
read services. It physically cannot send write-by-identifier, IO control, routine
control, ECU reset, or any transfer/flash service — a whitelist wraps the send path and
raises rather than transmitting. Reads can't brick an ECU. All the danger lives in the
services I've made unreachable.

**I'm not publishing the channel map.** The framing, the checksum, the method, and the
mistakes are all here, and that's the part with general value. The address-to-label
table is the vendor's ODX work product expressed as addresses; recovering it for my own
car is one thing, republishing it as a lookup table someone else can ship is another. So
that stays on my disk.

---

## What I'd tell someone starting this

1. **Capture the ground truth alongside the mystery.** The pcap alone is unsolvable in
   any reasonable time. The pcap *plus* the vendor's own CSV of the same fifteen seconds
   turns it into arithmetic.
2. **Put a fiducial in on purpose.** One clean rev, in both files, is worth more than an
   hour of driving. Give yourself an unambiguous event to align on.
3. **Guess the lazy checksum first.** Embedded firmware written to a deadline uses a
   sum. Test it across the entire capture before you go looking for CRC polynomials — and
   once it holds, you have an oracle for everything downstream.
4. **Never fit a signed channel.** The bug in this post cost me three weeks of wrong
   data on the single most important channel, and it produced output that looked
   completely healthy the entire time.
5. **Test that channels *move*, not that they're populated.** "Not null" is not
   "correct," and the gap between those two is where silent data corruption lives.
6. **Know when the data is gone.** I could invert one broken channel and not the other,
   and the honest move on the second one was a documented gap rather than a plausible
   guess. An unsupported hypothesis in your dataset is worse than a hole, because holes
   don't get built on.

---

*Written up from the working notes of a personal project. Happy to answer questions
about the capture setup or the decode.*
